#tls #x509 #san #certificates #k3s #kubernetes

# TLS SANs

The note behind this error:

```text
Unable to connect to the server: tls: failed to verify certificate:
x509: certificate is valid for 10.43.0.1, 127.0.0.1, 172.18.0.2, ...,
not 192.168.0.2
```

This is **not** a networking failure. The TCP connection succeeded, the handshake happened, a real certificate came back — it just isn't valid for the name that was dialed. Worked example throughout is the k3s API server under k3d (**01 - K3d lab cluster on pi-hole**), but nothing here is specific to Kubernetes.

## What a SAN is

A TLS certificate carries a list of identities it is allowed to speak for — the Subject Alternative Name extension. It holds two kinds of entry, DNS names and IP addresses, kept in separate buckets:

```text
X509v3 Subject Alternative Name:
    DNS:localhost, DNS:kubernetes.default.svc, IP Address:127.0.0.1, IP Address:10.43.0.1
```

## The two checks a client makes

When a TLS client opens a connection it performs two independent checks:

| # | Check | Answered by |
|---|---|---|
| 1 | Is this cert signed by a CA I trust? | the trust store — in kubeconfig, `certificate-authority-data` |
| 2 | Does the address I dialed appear in this cert's SAN list? | the SAN extension |

Both must pass. Check 2 is the one that bites, because check 1 usually succeeds — the CA is the cluster's own and it's right there in the kubeconfig.

**The Common Name field is not a fallback.** Historically a CN could stand in for a SAN. Go — which both `kubectl` and the API server are written in — stopped honouring CN entirely in **Go 1.15**. The SAN list is the only thing that counts now. A cert with a perfect CN and an empty SAN list matches nothing.

## IP SANs and DNS SANs are not interchangeable

Whatever string sits in the client's address field is what gets matched:

- dialing `https://192.168.0.2:6443` → needs an **IP** SAN for `192.168.0.2`
- dialing `https://pi-hole.local:6443` → needs a **DNS** SAN for `pi-hole.local`

A DNS SAN does nothing for a connection dialed by IP, even when that name resolves to exactly that IP. Resolution happens before the TLS check and leaves no trace in it — the client compares the literal string it was given.

k3s sorts each `--tls-san` value into the right bucket by parsing it, so pass both forms and let it sort them. Cheap insurance: it costs nothing at creation time, and it means the kubeconfig can later switch between IP and name without touching the cluster.

## Wildcards cover exactly one label

`DNS:*.example.com` matches `api.example.com`, but **not** `example.com` itself and **not** `a.b.example.com`. A wildcard stands for one label, no more and no fewer. Cover the apex by listing it as its own SAN entry.

Wildcards also apply to DNS names only — there is no such thing as a wildcard IP SAN.

## Certificates are minted once

The SAN list is baked in at generation time. Deciding six labs from now that you'd rather use a hostname means the existing certificate doesn't have it — and adding the flag to a running server changes nothing, because it finds a certificate already on disk and reuses it.

Fixing after the fact means deleting the cert and key so they get regenerated. For k3s, inside the server container:

```bash
rm /var/lib/rancher/k3s/server/tls/serving-kube-apiserver.crt \
   /var/lib/rancher/k3s/server/tls/serving-kube-apiserver.key
# then restart k3s
```

Doable, but recreating the cluster is usually less fuss. That is the argument for **listing every address you might plausibly use, up front** — LAN IP, hostname, mDNS name, VPN address, whatever the future looks like.

## Worked example: the k3s API server under k3d

k3s generates a self-signed CA and an API server certificate on first boot, auto-populating the SAN list with addresses it can work out for itself: `127.0.0.1`, `localhost`, the in-cluster service IP (`10.43.0.1`), the `kubernetes.default.svc` names — and the node's own IP.

That last one is the catch. Under k3d the "node" is a Docker container. Its IP is on the k3d bridge network, here `172.18.0.2`. The server has no idea it's reachable from the LAN as `192.168.0.2`; that address belongs to the *host*, not the container, and nothing inside the cluster ever sees it.

Publishing the port doesn't help. `--api-port 0.0.0.0:6443` fixes *reachability* and leaves *identity* broken — the two are separate problems and each needs its own flag.

Nor does the load balancer. k3d parks an nginx container in front of the servers, but it is a stream (TCP) proxy: it forwards bytes without terminating TLS. The certificate the client validates is the API server's own, so the proxy cannot paper over a SAN gap.

`--tls-san` simply appends an address to the list before the cert is minted:

```bash
k3d cluster create argocd \
  --k3s-arg "--tls-san=192.168.0.2@server:0" \
  --k3s-arg "--tls-san=pi-hole.local@server:0"
```

### Where the equivalent flag lives elsewhere

The principle is universal; only the flag name changes.

| Stack | Flag |
|---|---|
| k3s / k3d | `--tls-san` (repeatable) |
| kubeadm | `--apiserver-cert-extra-sans`, or `apiServer.certSANs` in the config |
| kube-apiserver directly | whatever minted `--tls-cert-file` — SANs come from your CSR |
| openssl / cfssl | `subjectAltName` in the CSR extensions |

## Reading a certificate

Read the SAN list off the wire rather than inferring it from whether the client happens to work:

| Source | Command |
|---|---|
| A live server | `openssl s_client -connect host:port </dev/null 2>/dev/null \| openssl x509 -noout -ext subjectAltName` |
| A file on disk | `openssl x509 -in cert.pem -noout -ext subjectAltName` |
| A specific name via SNI | add `-servername name.example.com` to `s_client` |
| Full detail | swap `-ext subjectAltName` for `-text` |

Confirmed on this cluster:

```bash
openssl s_client -connect 192.168.0.2:6443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName
```

```text
subject=O=k3s, CN=k3s
X509v3 Subject Alternative Name:
    DNS:k3d-argocd-server-0, DNS:k3d-argocd-serverlb, DNS:kubernetes,
    DNS:kubernetes.default, DNS:kubernetes.default.svc,
    DNS:kubernetes.default.svc.cluster.local, DNS:localhost, DNS:pi-hole.local,
    IP Address:0.0.0.0, IP Address:10.43.0.1, IP Address:127.0.0.1,
    IP Address:172.18.0.2, IP Address:192.168.0.2, IP Address:0:0:0:0:0:0:0:1
```

Both custom SANs landed: `IP Address:192.168.0.2` and `DNS:pi-hole.local`. Note also `CN=k3s` — meaningless as an identity, exactly as described above.

**Use `-servername` when one address serves many certificates.** Without it, `s_client` sends no SNI and the server returns its default certificate, which may not be the one you're debugging. Irrelevant for a single-cert API server; it changes the answer entirely on an ingress controller or a shared reverse proxy.

## The escape hatch, and why to skip it

All of this can be sidestepped with `insecure-skip-tls-verify: true` in the kubeconfig, or `k3d kubeconfig get --insecure`. It works.

It also disables **check 1 along with check 2** — the client stops verifying the CA signature entirely, not just the address match. Anything on the network path that can intercept the port could then present its own certificate and read an admin-credentialed session. Low probability on a home LAN, but `--tls-san` is one flag at creation time, so there is no reason to trade it away.

The general shape of that trade is worth remembering: **the two checks are not separately switchable.** Every "skip verification" flag in every TLS client turns off identity *and* trust together. There is no option that keeps the CA check while relaxing the name check.

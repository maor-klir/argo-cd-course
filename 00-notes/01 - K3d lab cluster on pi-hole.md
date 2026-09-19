#k3d #k3s #argocd #docker #ufw

# K3d lab cluster on a pi-hole

A k3d cluster on an always-on home server, driven with `kubectl` from the laptop.  
Built for the Argo CD course labs, which need only port-forward, a single cluster and outbound git access — no persistent volumes, no ingress, no LoadBalancer.

Companion notes: **02 - TLS SANs** for why `--tls-san` is mandatory here, **03 - Managing kubeconfigs** for how the context reaches the laptop.

## What k3d actually creates

k3d is not a Kubernetes distribution — it's a launcher that runs **k3s inside Docker containers**, one per node:

| Container | Runs | Holds |
|---|---|---|
| `k3d-argocd-server-0` | `k3s server` | API server, scheduler, controller-manager, datastore |
| `k3d-argocd-agent-0`, `-agent-1` | `k3s agent` | kubelet and kube-proxy — workloads only |
| `k3d-argocd-serverlb` | nginx | TCP proxy in front of the servers; where the published port lands |

Two consequences drive most of what follows:

- **Nodes carry container IPs, not host IPs:** they sit on the k3d bridge network (`172.18.0.0/16`). Nothing inside the cluster knows the host answers to `192.168.0.2` on the LAN — which is why the API certificate needs help.
- **The serverlb is a stream proxy:** it forwards TCP without terminating TLS, so it cannot paper over a certificate problem; the cert a client validates is the API server's own.

The server/agent split also explains why arguments need a node filter: `k3s server` and `k3s agent` are different subcommands accepting different flags.

## Host prerequisites

| Check | Why it matters | On this host |
|---|---|---|
| A free port for the API | k3d publishes it from the host | 6443 free — Pi-hole owns 53/80/443 |
| cgroup v2 with the `memory` controller | kubelet won't start without it | present |
| Docker subnet vs. LAN subnet | overlapping ranges break routing | docker0 `172.17/16`, k3d `172.18/16`, LAN `192.168.0/24` — no overlap |
| `/etc/resolv.conf` has a routable resolver | CoreDNS inherits it | `1.1.1.1` / `1.0.0.1` |
| Swap | kubelet refuses to run with swap by default | on (4 GB) — k3s sets `fail-swap-on=false`, so it's fine |

**The DNS check is the one to take seriously on a DNS server:**  
The classic k3s-on-a-resolver failure is CoreDNS inheriting `127.0.0.53` from systemd-resolved's stub listener.  
Inside a container that loopback address is the *container's own*, so every cluster lookup fails with nothing obviously wrong in the config.  
Pi-hole's installer disables the stub listener, so `/etc/resolv.conf` here hands out real routable addresses and CoreDNS works untouched. Image pulls bypass Pi-hole filtering as a side effect.

## Create the cluster

### Config file

For a cluster meant to stay up, encode it in a file rather than a shell command to be reconstructed in six months.  
**Validated against k3d 5.9.0** — round-trips cleanly through `k3d config migrate`:

```yaml
# argocd-lab.yaml
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: argocd
servers: 1
agents: 2
kubeAPI:
  hostIP: "0.0.0.0"      # replaces --api-port
  hostPort: "6443"
options:
  k3s:
    extraArgs:
      - arg: --tls-san=192.168.0.2
        nodeFilters:
          - server:*
      - arg: --tls-san=pi-hole.local
        nodeFilters:
          - server:*
      - arg: --disable=traefik
        nodeFilters:
          - server:*
```

```bash
k3d cluster create --config argocd-lab.yaml
```

The `arg` / `nodeFilters` split is explicit instead of encoded in an `@`, and there is no shell quoting to get wrong.

### CLI equivalent

What was run first, and what the config file replaces:

```bash
k3d cluster create argocd \
  --api-port 0.0.0.0:6443 \
  --k3s-arg "--tls-san=192.168.0.2@server:0" \
  --k3s-arg "--tls-san=pi-hole.local@server:0" \
  --k3s-arg "--disable=traefik@server:0" \
  --agents 2
```

That flag has three grammars nested inside each other — see *The `--k3s-arg` syntax* in Reference.

### Why these flags

**`--api-port 0.0.0.0:6443`** publishes the API on every host interface instead of loopback only. It fixes *reachability* and leaves *identity* broken.

**The two `--tls-san` values** fix identity. k3s mints its serving certificate on first boot from addresses it can work out for itself, and the host's LAN address isn't one of them. IP and DNS SANs are not interchangeable, so both forms are passed — full reasoning in **02 - TLS SANs**. Certificates are minted **once**: adding a SAN to a running cluster does nothing, so list every address we might plausibly use up front.

**`--disable=traefik`** because Pi-hole owns host 80/443. k3d doesn't map those unless asked, so there's no hard conflict — but disabling it removes the trap and keeps the Argo CD resource tree free of an app we never deployed.

> **Never add `-p 80:80@loadbalancer` to this cluster.** That *would* collide with the Pi-hole UI.

## Connect kubectl

```bash
ssh pi-hole '/home/linuxbrew/.linuxbrew/bin/k3d kubeconfig get argocd' > /tmp/k3d-argocd.yaml
sed -i 's|https://0.0.0.0:6443|https://192.168.0.2:6443|' /tmp/k3d-argocd.yaml
install -m 600 /tmp/k3d-argocd.yaml ~/.kube/cluster_configs/k3d-argocd.yaml
~/.kube/merge.sh
```

Three things are load-bearing:

- **The absolute path** — brew's PATH does not exist under `ssh host 'cmd'`, see Reference.
- **The `sed`** — k3d writes `https://0.0.0.0:6443`, which is meaningless from another machine.
- **Temp file, then `install`** — `>` creates the destination *before* the command runs, so a failed fetch leaves a 0-byte file that silently pollutes the next merge. `install` copies a file and sets its permissions in one step.

Full workflow, including the merge script: **03 - Managing kubeconfigs**.

## Verify

```bash
kubectl --context=k3d-argocd get nodes
```

```text
NAME                  STATUS   ROLES           AGE   VERSION
k3d-argocd-agent-0    Ready    <none>          32m   v1.35.5+k3s1
k3d-argocd-agent-1    Ready    <none>          32m   v1.35.5+k3s1
k3d-argocd-server-0   Ready    control-plane   32m   v1.35.5+k3s1
```

```bash
kubectl --context=k3d-argocd get --raw /readyz     # -> ok
```

Then read the certificate's SAN list straight off the wire, rather than inferring it from whether kubectl happens to work:

```bash
openssl s_client -connect 192.168.0.2:6443 </dev/null 2>/dev/null \
  | openssl x509 -noout -ext subjectAltName
```

Both custom SANs landed — `IP Address:192.168.0.2` and `DNS:pi-hole.local`. Full output in **02 - TLS SANs**.

## Keep it running

The box is meant to stay up, so the cluster should come back on its own. Check what Docker will do:

```bash
ssh pi-hole 'docker inspect -f "{{.Name}} {{.HostConfig.RestartPolicy.Name}}" $(docker ps -aq --filter name=k3d-argocd)'
```

If those aren't `unless-stopped`, add a boot unit:

```ini
[Unit]
Description=k3d argocd cluster
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
User=maor
ExecStart=/home/linuxbrew/.linuxbrew/bin/k3d cluster start argocd

[Install]
WantedBy=multi-user.target
```

> Note the absolute brew path — systemd has even less PATH than `ssh host 'cmd'`.


## Reference

### The `--k3s-arg` syntax

```bash
--k3s-arg "--tls-san=192.168.0.2@server:0"
└────┬───┘ └──────┬───────┘ └────┬────┘
     │            │              │
     │            │              └─ k3d node filter — which containers get it (stripped)
     │            └─ the actual k3s flag, passed through verbatim
     └─ k3d passthrough flag
```

Three syntaxes nested inside each other, plus shell quoting.

**Layer 1 — `--k3s-arg` is a passthrough:** k3d's own CLI covers k3d-level concerns: containers, networks, port mappings. `--tls-san` isn't one of those; it belongs to the k3s process inside the node container. **k3d 5.9.0 has no `--tls-san` flag of its own** (verified against its full `cluster create --help`), so `--k3s-arg` is the generic "append this string to the k3s command line" passthrough. Same mechanism as `--disable=traefik`.

**Layer 2 — bind flag and value with `=`, not a space:** written as `--tls-san 192.168.0.2@server:0` it breaks: k3d hands k3s a single argv entry with a space inside it rather than a flag followed by its value, and k3s sees one unrecognizable token.

The constraint is specific to `--k3s-arg`, not to k3s. k3s itself accepts the space form when the two are genuinely separate argv elements — visible in the container's own command line, where k3d appends its own SANs that way:

```json
["server","--tls-san=192.168.0.2","--tls-san=pi-hole.local","--disable=traefik",
 "--tls-san","0.0.0.0","--tls-san","k3d-argocd-serverlb"]
```

The first three are ours, passed via `--k3s-arg` and so in `=` form. The last two are k3d's own additions as separate elements. **Inside `--k3s-arg`, always use `=`.**

**Layer 3 — `@server:0` is the node filter, and it's mandatory:** this part never reaches k3s; k3d splits it off and uses it to decide which node containers receive the argument. It matters because the containers aren't interchangeable — `--tls-san` configures the API server's certificate, which exists only on servers. Handing it to an agent means passing an unrecognized flag to a different subcommand.

| Filter | Meaning | Status |
|---|---|---|
| `server:0` | first server node only | verified — used here |
| `server:*` | all server nodes | documented |
| `agent:1` | a specific agent by index | documented |
| `loadbalancer` | the serverlb container | documented |
| `all` | every node | documented |
| `agent:0,1` | comma-separated index list | **unverified** — not in k3d docs, don't rely on it |

Multi-filter form is `ARG@NODEFILTER[;@NODEFILTER]`, straight from `k3d cluster create --help`. Note the `@` repeats after the semicolon: `"--tls-san=192.168.0.2@server:0;@server:1"`.

This cluster has one server, so `server:0` and `server:*` are equivalent. Prefer `server:*` — if we ever add servers for an HA experiment, the SAN follows automatically instead of landing on only the first.

**Why two `--k3s-arg` flags instead of one:** each carries exactly one k3s argument. `--tls-san` is repeatable on the k3s side, one occurrence per SAN, so two SANs means two occurrences means two `--k3s-arg` flags.

**Why the quotes:** `@` and `:` aren't special to bash, so quotes are decorative in this exact string. They stop being decorative with `server:*` (bash globs `*`) or the multi-filter form (`;` is a command separator — unquoted, bash would try to run `@server:1`). Always quote.

Verify the filter was consumed rather than passed through:

```bash
ssh pi-hole "docker inspect k3d-argocd-server-0 -f '{{json .Config.Cmd}}'"
```

`--tls-san=192.168.0.2` appears as its own array element with **no `@server:0` attached** — proof k3d stripped it. Confirmed in the output above.

### brew is invisible to `ssh host 'cmd'`

```bash
$ ssh pi-hole 'k3d kubeconfig get argocd'
bash: line 1: k3d: command not found
$ ssh pi-hole            # then interactively
eaor@pi-hole:~$ k3d cluster ls     # works fine
```

k3d is a **Homebrew** install at `/home/linuxbrew/.linuxbrew/bin/k3d`. Brew's `shellenv` is evaluated in the interactive rc only, so that directory is missing from the non-interactive PATH that `ssh host 'cmd'` gets. Same host, same user, different PATH. Fix: absolute path in ssh one-liners.

Three follow-on traps this exposed:

- **`command -v` over ssh silently under-reports:** an early survey concluded "k3d NOT_INSTALLED" on this host for exactly this reason. On checking, brew shadows only `k3d` here — `kubectl` v1.35.5 and `helm` v3.20.0 are identical interactively and non-interactively — but the failure mode is silent, so re-check anything brew might own with `bash -ic`.
- **`find / -maxdepth 4 -name k3d -type f` missed it twice over:** the path is depth 5, and it's a symlink into `Cellar` so `-type f` excludes it. Use `-maxdepth 6` and drop `-type f` when hunting for a binary.
- **The failed fetch still created a file:** `>` truncates its target before the command runs, so a 0-byte `k3d-argocd.yaml` landed in `cluster_configs/` and would have polluted the next merge — hence the temp-file dance in *Connect kubectl* above.

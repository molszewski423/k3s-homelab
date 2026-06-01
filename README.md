# k3s-homelab

A complete 3-node k3s homelab built over a weekend (May 2026), running two production namespaces: a clinical AI platform with RTX 5060 Ti GPU inference, and a 24-service AI agency stack migrated live from Podman.

---

## What Was Built

Starting from three bare Linux machines on a home LAN, the following was operational by end of the weekend:

- k3s v1.35 cluster, 3 nodes joined and healthy
- RTX 5060 Ti GPU exposed to Kubernetes via NVIDIA device plugin + RuntimeClass
- Ollama running in a GPU pod, 5 models loaded and serving inference
- Two clinical AI Streamlit apps with Traefik ingress (`pv.lan`, `ams.lan`)
- Argus Discord bot connected to pv-workbench backend
- GitLab CI/CD building and deploying images on push to main
- 24 agency services migrated from Podman systemd quadlets to k3s - live, no data loss
- PostgreSQL 16 data preserved via hostPath PVCs (zero dump/restore)
- Cloudflare tunnel routing `ringcatch.io` and `dashboard.ringcatch.io` through the cluster

---

## Architecture

![Architecture](docs/architecture.png)

---

## Why k3s

| Option | Reason rejected |
|---|---|
| Podman systemd quadlets | No rolling updates, ingress, or health-based scheduling. UID remapping breaks volume permissions across nodes. Works for single machine; brittle across 3. |
| Docker Compose / Swarm | Swarm is in maintenance mode. Compose is single-host. |
| kubeadm (full k8s) | etcd overhead, complex bootstrapping, overkill for a homelab. |
| k3s | Single binary, embeds containerd + Traefik + CoreDNS + local-path-provisioner. CNCF-conformant. Joins a node in under 2 minutes. Production-grade Kubernetes API. |

---

## Phase 1 - Cluster Setup

### Control plane (MikePC)

```bash
curl -sfL https://get.k3s.io | sh -
sudo cat /var/lib/rancher/k3s/server/node-token   # save this for workers
```

### Workers (archbox, MikeInspiron)

```bash
# Fish shell requires `env` for inline variable assignment
curl -sfL https://get.k3s.io | env K3S_URL=https://192.168.4.54:6443 K3S_TOKEN=<token> sh -
```

### NVIDIA GPU (MikePC)

k3s **auto-configures the NVIDIA container runtime** - no manual `config.toml.tmpl` needed. Install the toolkit, configure it, restart k3s:

```bash
sudo apt install nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=containerd
sudo systemctl restart k3s
```

Deploy the device plugin:

```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/main/deployments/static/nvidia-device-plugin.yml
```

Create a RuntimeClass (required - resource limits alone are not enough):

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
```

GPU pods must declare both:

```yaml
spec:
  runtimeClassName: nvidia
  containers:
    - resources:
        limits:
          nvidia.com/gpu: "1"
```

---

## Phase 2 - Clinical AI Platform (namespace: ai)

All pods use `nodeSelector: kubernetes.io/hostname: mikepc` to pin to the GPU node. Manifests: [`homelab-infra/k8s/`](https://gitlab.com/molszewski423/homelab-infra).

**Ollama** runs as a GPU Deployment with a PVC for model storage. Models are pulled via `kubectl exec`:

```bash
kubectl exec -n ai deploy/ollama -- ollama pull gemma4:26b
```

**pv-workbench** and **ams-intelligence** are built by GitLab CI on every push to `main`/`master`, pushed to `registry.gitlab.com/molszewski423/<app>:latest`, and rolled out via `kubectl rollout restart`.

**Traefik ingress** (k3s built-in) routes `pv.lan` and `ams.lan` via LAN `/etc/hosts`:

```
192.168.4.54  pv.lan ams.lan
```

**argus-bot** is a second Deployment using the same pv-workbench image with CMD overridden to `python3 src/argus_bot.py`. Shares ChromaDB and output PVCs with the Streamlit pod.

---

## Phase 3 - Agency Migration (namespace: agency)

### Background

RingCatch ran as 24 Podman rootless systemd quadlets on archbox - one `.container` unit file per service, environment from `~/agency/.env`. Worked for a single machine. No rolling updates, no health routing, rebuilds required manual restarts.

### Approach

Each Podman unit → Kubernetes Deployment + Service. All 24 pinned to archbox via `nodeSelector`. PostgreSQL uses a hostPath PV pointing at the existing Podman volume data directory - no dump/restore required.

Secrets: `~/agency/.env` → `kubectl create secret generic agency-env --from-env-file=.env`

### Challenges Solved

**Podman UID mapping breaks hostPath volume permissions**

Podman rootless containers remap UIDs. Files written by Podman containers appear as uid `100999+` on the host. k3s/containerd runs containers with direct UIDs (no remapping). The container could not read its own data.

Fix: chown the data directories to the container's actual runtime UID before mounting as hostPath.

```bash
# n8n runs as node (uid 1000) in containerd
sudo chown -R 1000:1000 /path/to/n8n-data/
```

**localhost refs break when a Podman pod splits to k3s pods**

Inside a Podman pod, all containers share a network namespace - `localhost:8080` reaches any container. In k3s, each Deployment is an isolated pod. Every `localhost`/`127.0.0.1` service reference must become a Kubernetes DNS name.

```
# Before
ORCHESTRATOR_URL=http://127.0.0.1:8109

# After
ORCHESTRATOR_URL=http://agency-orchestrator:8109
```

This broke the command dashboard health checks (13 hardcoded `127.0.0.1` URLs) and the Discord bot alert path - all had to be updated to k8s service names.

**hostPort + rolling updates conflict**

Rolling updates create the new pod before terminating the old one. Two pods cannot bind the same hostPort simultaneously - the new pod stays `Pending`. Fix: either avoid hostPort (use Traefik ingress + ClusterIP), or manually delete the old pod before a rolling update.

**Local images not in k3s containerd**

Agency images were built locally with Podman and not in any registry. k3s/containerd cannot see Podman's image store. Fix:

```bash
# On archbox - export from Podman, import into k3s
podman save localhost/agency-orchestrator:latest | sudo k3s ctr -n k8s.io images import -
```

Manifests use `imagePullPolicy: Never`. Long-term: push to GitLab registry, let CI handle it.

**n8n encryption key mismatch**

The n8n `config` file had stored a literal placeholder `{N8N_ENCRYPTION_KEY}` as the encryption key (env var was never expanded in Podman). k3s injected the real value from the secret, causing a mismatch. Fix: delete the broken config file and let n8n regenerate it.

**Cloudflare tunnel initContainer (distroless image)**

The tunnel purge script used `/bin/sh` via an initContainer on the cloudflared image. cloudflared is distroless - no shell available. Fix: remove the initContainer and run the purge directly via the Cloudflare API instead.

---

## Phase 4 - Third Node (MikeInspiron)

Dell Inspiron joining as a 24/7 worker with lid closed. Sleep prevention required:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Hyprland configured for lid-close → display off, lid-open → display on via `bindl` - systemd-logind set to ignore lid events. The machine stays logged in for occasional local use.

Join command (Fish shell - note `env` syntax):

```bash
# Set token first - Fish does not support KEY=VAL cmd syntax
set TOKEN <node-token>
curl -sfL https://get.k3s.io | sudo env K3S_URL=https://192.168.4.54:6443 K3S_TOKEN=$TOKEN sh -
```

---

## Key Lessons

### k3s auto-configures NVIDIA runtime - no config.toml.tmpl needed

Many guides instruct you to write a `config.toml.tmpl` for k3s containerd. This is unnecessary and breaks things if done manually. k3s generates its containerd config at startup. Install `nvidia-container-toolkit`, run `nvidia-ctk runtime configure --runtime=containerd`, restart k3s.

### RuntimeClass + resource limits are both required for GPU pods

`nvidia.com/gpu: "1"` in resource limits is necessary but not sufficient. Without `runtimeClassName: nvidia`, the pod schedules but the GPU is inaccessible. Both are required.

### hostPort conflicts with rolling updates - avoid or manage manually

hostPort prevents two pods from running simultaneously on the same node. Rolling updates require overlap. Either use Traefik ingress + ClusterIP (preferred), or `kubectl delete pod` the old pod manually before updating.

### Podman UID remapping breaks hostPath volumes in k3s

Rootless Podman remaps container UIDs to subUID ranges on the host. k3s runs containers with direct UIDs. Files written under Podman appear as high UIDs to k3s containers. Always `chown` host data directories to the container's actual runtime UID when migrating stateful Podman workloads to k3s.

### localhost refs break across k8s pod boundaries

A Podman pod is one network namespace shared by all containers. k3s Deployments are isolated pods. Every internal `localhost:PORT` must become `<service-name>:PORT` (Kubernetes DNS resolves service names within the same namespace automatically).

### imagePullPolicy: Never + k3s ctr -n k8s.io images import for local images

k3s's containerd image store is separate from Podman and Docker. Images must be imported explicitly with the `-n k8s.io` namespace flag. Without it, images land in the `default` containerd namespace where kubelet cannot find them.

### Fish shell: use `env KEY=VAL cmd`, not `KEY=VAL cmd`

Fish does not support the bash/POSIX inline variable syntax. When running k3s commands from a Fish session:

```fish
# Wrong (Fish syntax error)
K3S_TOKEN=abc sh -

# Correct
env K3S_TOKEN=abc sh -
```

---

## Future: AWS Hybrid

| Workload | Target |
|---|---|
| Stateless landing pages, webhook handlers | AWS EC2 t2.micro / Lambda (free tier) |
| LLM inference | On-prem MikePC - RTX 5060 Ti, cloud GPU is too expensive |
| Stateful databases, agency services | On-prem archbox - data sovereignty, zero egress cost |
| Clinical AI | On-prem - HIPAA sensitivity, GPU dependency |

Tailscale will bridge on-prem and AWS nodes. Same k3s manifests, new node pool added to the cluster.

---

## Related Repos

| Repo | Description |
|---|---|
| [homelab-infra](https://gitlab.com/molszewski423/homelab-infra) | All k3s manifests (ai + agency namespaces) |
| [pv-workbench](https://gitlab.com/molszewski423/pv-workbench) | Pharmacovigilance platform |
| [ams-intelligence](https://gitlab.com/molszewski423/ams-intelligence) | Antimicrobial stewardship platform |
| [ringcatch-agency](https://gitlab.com/molszewski423/ringcatch-agency) | RingCatch AI agency - 24 k3s services |
| [dotfiles](https://gitlab.com/molszewski423/dotfiles) | Fish, Hyprland, Kitty, Neovim |

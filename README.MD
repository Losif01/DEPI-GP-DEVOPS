
# Nextcloud Private Infrastructure 

## Overview

This repository defines the current bare-metal/containerized baseline for our private Nextcloud infrastructure. The current environment is running on an Ubuntu host using Docker Compose, prometheus and grafana were used for monitoring, using the industry standard dashboard `Node exporter full`

## 1. Current Architecture & Tech Stack

The current stack is a declarative microservice architecture, intentionally avoiding the Nextcloud "All-In-One" (AIO) image to allow for granular control and an easier migration to K8s pods.

- **Host OS:** Ubuntu Server
    
- **Container Runtime:** Docker Engine
    
- **Orchestration:** Docker Compose (Targeting migration to K8s)
    
- **Application Layer:** Nextcloud (PHP-FPM / Apache)
    
- **Database Layer:** MariaDB
    
- **Network / Ingress:** Tailscale (WireGuard mesh VPN) routing directly to the host on port 8080.
	
- **Monitoring:** there exists stacks for both prometheus and grafana at port 3000.

## 2. Producing the Current Environment

To spin up the existing baseline for testing or development, follow this standard workflow on the Ubuntu host.

### Step 2.1: Infrastructure Deployment

Navigate to the directory containing the `docker-compose.yml` and bring up the stack:
*that's it.*

```bash 
docker compose up -d
```

_Wait ~60 seconds for MariaDB to initialize and the application container to bind._

### Step 2.2: Network Overlay Configuration

The infrastructure currently bypasses traditional reverse proxies (like Nginx/Traefik) and public domains. All ingress is handled via a Tailscale subnet.

1. Install and authenticate Tailscale on the host:
    
    
    ``` bash
    curl -fsSL https://tailscale.com/install.sh | sh
    sudo tailscale up
    ```
    
2. Retrieve the host's Tailscale IP that was sent to you in mail:(`100.X.X.X`):

### Step 2.3: Application Configuration (Trusted Domains)

By default, the Nextcloud container will block traffic originating from the Tailscale interface. You must inject the VPN IP into the application's config.

Run the following command against the active container, replacing `100.X.X.X` with the server's Tailscale IP:

```bash
docker compose exec --user www-data app php occ config:system:set trusted_domains 1 --value="100.X.X.X"
```

## 3. Storage and State Management

For the upcoming Kubernetes migration, note the following persistence targets currently handled by Docker volumes. These will need to be mapped to Persistent Volumes (PV) and Persistent Volume Claims (PVC) in the cluster:

- **Database Data:** MariaDB storage (critical for user accounts and file metadata).
    
- **Nextcloud HTML/Config:** Contains the `config.php` file and installed app configurations.
    
- **Nextcloud Data:** The primary storage pool containing actual user files and the audit `nextcloud.log`.
    

# Cloud-Stack — Kubernetes Infrastructure (Phases 1–3)

Automated, self-healing Nextcloud platform backed by PostgreSQL on Kubernetes (k3s),
with nodes connected over Tailscale.

This document covers **Phase 1 (cluster bootstrap), Phase 2 (persistent storage),
and Phase 3 (PostgreSQL)**. Nextcloud (Phase 4), tailnet exposure (Phase 5) and
monitoring (Phase 6) are documented separately.

> Scope of this layer: **Kubernetes orchestration** — standing up the cluster,
> persistent storage, and the database workload. Application-level configuration
> (Nextcloud settings, DB schema design) is a separate teammate responsibility.

---

## 1. Environment

| Role | Hostname | OS | LAN IP | Tailscale IP | Tailnet name | Account |
|---|---|---|---|---|---|---|
| **Server** (k3s control-plane) | HomeYFH | Ubuntu 24.04.4 LTS (kernel 6.8.0-124) | 192.168.1.6 | 100.70.23.17 | homeyfh.tail0c124d.ts.net | skillissuecmaster@ (shared in) |
| **Workstation VM** (kubectl) | work | Ubuntu | — | 100.108.198.79 | — | rokiaossama@ |

- **Kubernetes distribution:** k3s `v1.35.5+k3s1` (single node, sqlite datastore)
- **Container runtime:** containerd (bundled with k3s)
- **CNI / storage:** flannel + `local-path` provisioner (both k3s built-ins)
- **Project working directory on server:** `~/cloud-stack/`

```
cloud-stack/
├── postgres/        # PostgreSQL StatefulSet + Service (Phase 3)
├── nextcloud/       # (Phase 4 — not in this doc)
├── monitoring/      # (Phase 6 — not in this doc)
└── expose/          # (Phase 5 — not in this doc)
```

---

## 2. Ports

| Port | Component | Scope | Notes |
|---|---|---|---|
| **6443** | Kubernetes API server (k3s) | Reached over Tailscale for remote `kubectl` | Must be allowed by the tailnet ACL for cross-account access |
| **5432** | PostgreSQL | In-cluster only (headless Service, `clusterIP: None`) | Not exposed outside the cluster |

---

## 3. Phase 1 — Cluster Bootstrap

### 3.1 Install k3s on the server

Run **on the server (HomeYFH)**. `--tls-san` bakes the Tailscale IP into the API
certificate so the cluster can be reached remotely over Tailscale.

```bash
curl -sfL https://get.k3s.io | sh -s - --tls-san=100.70.23.17
```

Verify:

```bash
sudo k3s kubectl get nodes
# NAME      STATUS   ROLES           AGE   VERSION
# homeyfh   Ready    control-plane   ...   v1.35.5+k3s1
```

### 3.2 Make `kubectl` usable without sudo (on the server)

The kubeconfig at `/etc/rancher/k3s/k3s.yaml` is root-owned. Copy it to your user:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
unset KUBECONFIG
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc

kubectl get nodes -o wide
```

> The copy on the server points at `https://127.0.0.1:6443` — correct, because it is
> the server talking to itself. Do **not** rewrite that IP on the server copy.

---

## 4. Phase 2 + Phase 3 — Storage & PostgreSQL

Persistent storage and PostgreSQL are deployed together: the StatefulSet's
`volumeClaimTemplates` requests a `local-path` PersistentVolume, which gives data
that survives pod restarts and reschedules. On a single node, `local-path` is the
correct choice (no Longhorn needed; Longhorn would only be required for multi-node
replicated failover).

### 4.1 Namespace and credentials (Secret — nothing hardcoded)

```bash
cd ~/cloud-stack
mkdir -p postgres nextcloud monitoring expose

kubectl create namespace cloud

kubectl create secret generic postgres-secret \
  --namespace cloud \
  --from-literal=POSTGRES_DB=nextcloud \
  --from-literal=POSTGRES_USER=nextcloud \
  --from-literal=POSTGRES_PASSWORD="$(openssl rand -base64 20)"
```

Retrieve the generated password later if needed:

```bash
kubectl -n cloud get secret postgres-secret \
  -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d; echo
```

### 4.2 Manifest — `postgres/postgres.yaml`

Create the file with an editor (`nano postgres/postgres.yaml`) — pasting large YAML
via shell heredocs corrupts indentation, so use an editor.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: cloud
spec:
  clusterIP: None          # headless, for the StatefulSet
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: cloud
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgres-secret
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "nextcloud", "-d", "nextcloud"]
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "nextcloud", "-d", "nextcloud"]
            initialDelaySeconds: 30
            periodSeconds: 15
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-path
        resources:
          requests:
            storage: 5Gi
```

### 4.3 Apply and verify

```bash
kubectl apply -f postgres/postgres.yaml
kubectl -n cloud get pods -w        # wait for postgres-0 -> 1/1 Running, then Ctrl+C

kubectl -n cloud get pvc            # data-postgres-0 should be Bound (5Gi, local-path)

kubectl -n cloud exec -it postgres-0 -- pg_isready -U nextcloud -d nextcloud
# /var/run/postgresql:5432 - accepting connections
```

**Expected end state:**

```
NAME         READY   STATUS    RESTARTS   AGE
postgres-0   1/1     Running   0          ...

NAME              STATUS   VOLUME                 CAPACITY   STORAGECLASS
data-postgres-0   Bound    pvc-...                5Gi        local-path
```

---

## 5. Errors Encountered & Fixes

A faithful log of what broke during setup and how each was resolved.

### 5.1 `ssh@100.70.23.17: command not found`
**Cause:** wrong SSH syntax (a stray `@`, no username).
**Fix:** correct form is `ssh <user>@<host>`, e.g. `ssh hodhod@100.70.23.17`.

### 5.2 VM logged out of Tailscale
**Symptom:** `tailscale ping` returned `Logged out. Log in at: <url>`.
**Fix:**
```bash
sudo tailscale up      # then authenticate in browser
tailscale status       # confirm the VM appears on the tailnet
```

### 5.3 `kubectl` from the VM → `dial tcp 100.70.23.17:6443: i/o timeout`
**Symptom:** API unreachable from the workstation VM, even though:
- `tailscale ping 100.70.23.17` → **pong** (tailnet path OK)
- `sudo ufw status` → **inactive** (no host firewall)
- `sudo systemctl status k3s` → **active (running)**
- `sudo ss -tlnp | grep 6443` → listening on `*:6443`

So the server side was healthy; only **port 6443 specifically** was blocked.

**Root cause:** the server is **shared into** the tailnet from a different account
(`skillissuecmaster@`), while the VM is on `rokiaossama@`. Tailscale's policy layer
(not the host firewall) was dropping the connection. An attempted ACL rule:

```json
{ "action": "accept", "src": ["rokiaossama@"], "dst": ["100.70.23.17:6443"] }
```

did **not** resolve it (cross-account share + `src` matcher mismatch).

**Resolution adopted:** work directly on the server over SSH (all `kubectl apply`
steps run on the server itself — no remote access required).
**Permanent fix (recommended):** consolidate both machines onto **one** Tailscale
account so the server is *owned* rather than *shared* — then the default ACL applies
and 6443 works with no rule. See §6.

Diagnostic commands used:
```bash
tailscale status
tailscale ping 100.70.23.17
nc -vz 100.70.23.17 6443
sudo ufw status
sudo systemctl status k3s
sudo ss -tlnp | grep 6443
```

### 5.4 `permission denied` reading `/etc/rancher/k3s/k3s.yaml`
**Symptom:** `error loading config file ... permission denied` when running plain
`kubectl` as a non-root user.
**Cause:** the kubeconfig is root-owned.
**Fix:** copy it to the user and `chown` (see §3.2), or prefix everything with
`sudo k3s kubectl`.

### 5.5 Corrupted `postgres.yaml` / `No such file or directory`
**Symptom:** `postgres/postgres.yaml: No such file or directory`, and heredoc paste
produced garbled YAML.
**Cause:** the `postgres/` directory didn't exist yet, and the terminal reflowed a
`cat << EOF` heredoc.
**Fix:** `mkdir -p postgres` first, and create the file with `nano` instead of a
heredoc.

### 5.6 `postgres-0` stuck `Pending` → `ProvisioningFailed`
**Symptom:** pod Pending with no scheduling events; node was healthy
(`Taints: <none>`, `Unschedulable: false`, no Disk/Memory pressure).
**Real cause (from `kubectl get events -n cloud`):**
```
ProvisioningFailed: failed to provision volume with StorageClass "local-path":
  create process timeout after 120 seconds
helper-pod-create-pvc-...  ErrImagePull
  dial tcp: lookup registry-1.docker.io: Try again
```
The `local-path` provisioner spins up a helper pod to create the volume directory;
that helper couldn't pull its image due to **DNS failure on the node**. The PVC
self-recovered on retry once DNS briefly worked, and bound (5Gi). Diagnostics:
```bash
kubectl -n cloud get pvc
kubectl get storageclass
kubectl -n kube-system get pods | grep local-path
kubectl -n cloud describe pod postgres-0 | tail -20
kubectl get events -n cloud --sort-by=.lastTimestamp | tail -20
```

### 5.7 `ImagePullBackOff` on `postgres:16` — node DNS broken
**Symptom:** image pulls repeatedly failed:
```
dial tcp: lookup registry-1.docker.io: Try again
```
Even a manual `sudo k3s ctr images pull docker.io/library/postgres:16` failed the
same way → **persistent DNS problem**, not transient.

**Cause:** `systemd-resolved` was resolving through Tailscale MagicDNS, which was
intermittently failing to forward public lookups. `/etc/resolv.conf` showed the
stub `nameserver 127.0.0.53` and `search tail0c124d.ts.net`.

**Fix (the key unblocker for ALL image pulls):**
```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
echo -e '[Resolve]\nDNS=1.1.1.1 8.8.8.8\nFallbackDNS=8.8.4.4' \
  | sudo tee /etc/systemd/resolved.conf.d/dns.conf
sudo systemctl restart systemd-resolved

# verify
resolvectl status | grep -A2 'DNS Servers'
getent hosts registry-1.docker.io        # should now print IP addresses
sudo k3s ctr images pull docker.io/library/postgres:16
```
After this, the image pulled, `postgres-0` reached `1/1 Running`, and `pg_isready`
returned `accepting connections`.

> **Note:** this DNS flakiness recurs on every image pull until fixed. The
> `resolved.conf.d` drop-in above is the durable fix. If lookups still fail after it,
> the cause is Tailscale "Override local DNS" in the tailnet admin console.

---

## 6. Adding Another VM to This Setup

There are **two** different things "adding a VM" can mean. Pick the one you need.

### 6.1 Add a VM as a remote `kubectl` workstation (control the cluster remotely)

This is the workstation pattern — the VM does **not** run k3s; it only talks to the
API over Tailscale.

**On the new VM:**
```bash
# 1. Join the SAME Tailscale account/tailnet as the server (important — see §5.3)
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# 2. Install kubectl
sudo snap install kubectl --classic     # Ubuntu
# (Fedora: sudo dnf install -y kubectl)

# 3. Copy the kubeconfig from the server
mkdir -p ~/.kube
#   paste the contents of the server's ~/.kube/config (or /etc/rancher/k3s/k3s.yaml)
nano ~/.kube/config

# 4. Point it at the server's Tailscale IP (the file ships with 127.0.0.1)
sed -i 's|https://127.0.0.1:6443|https://100.70.23.17:6443|' ~/.kube/config

# 5. Test
kubectl get nodes -o wide
```

**Requirement:** port **6443** must be reachable from the VM over Tailscale. If the
server is *owned* on the same account, the default ACL allows this. If it is
*shared* from another account, you will hit the timeout in §5.3 — consolidate onto
one account, or have the tailnet owner permit `dst: ["<server-ts-ip>:6443"]`.

### 6.2 Add a VM/PC as a worker node (k3s agent — true multi-node)

This grows the cluster so pods can run on more than one machine.

**On the server**, get the join token:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

**On the new worker** (join the same tailnet first, as in §6.1 step 1), then:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://100.70.23.17:6443 \
  K3S_TOKEN="<paste-node-token>" sh -
```

**Back on the server**, confirm it joined:
```bash
kubectl get nodes -o wide        # the new node should appear as Ready, role <none>
```

**Multi-node storage caveat:** `local-path` is node-local. Once you have multiple
nodes, a stateful pod (PostgreSQL, Nextcloud data) that reschedules to a different
node will not find its data. Options:
- **Simple:** pin stateful pods to one node with a `nodeSelector`.
- **HA:** install **Longhorn** for replicated block storage across nodes (this is
  what actually backs a self-healing, fail-over availability target).

> For cross-node pods to communicate over Tailscale, the tailnet ACL must allow the
> pod CIDR (k3s default `10.42.0.0/16`). With the built-in k3s Tailscale integration
> (`--vpn-auth`), node IPs are replaced with tailnet IPs automatically.

---

## 7. Quick Reference

```bash
# cluster
kubectl get nodes -o wide
kubectl get pods -A

# this stack
kubectl -n cloud get pods
kubectl -n cloud get pvc
kubectl -n cloud get svc

# database health
kubectl -n cloud exec -it postgres-0 -- pg_isready -U nextcloud -d nextcloud

# logs / debug
kubectl -n cloud describe pod postgres-0
kubectl get events -n cloud --sort-by=.lastTimestamp | tail -20

# restart k3s (clears transient control-plane stalls)
sudo systemctl restart k3s
```

## 8. Status

| Phase | Scope | Status |
|---|---|---|
| 1 | k3s cluster bootstrap (single node, Tailscale-reachable) | ✅ Done |
| 2 | Persistent storage (local-path, PVC Bound, survives pod delete) | ✅ Done |
| 3 | PostgreSQL (StatefulSet, headless Service, Secret, probes) | ✅ Done |
| 4 | Nextcloud | ✅ Done |
| 5 | Tailnet exposure | ✅ Done |
| 6 | Prometheus + Grafana monitoring | ✅ Done |

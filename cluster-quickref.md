# ORYXSERVER Kubernetes Cluster — AI Deployment Quick Reference

## READ THIS FIRST

This is the quick reference guide for deploying applications to Christina's self-hosted Talos Kubernetes cluster. If you are an AI assistant helping Christina deploy, configure, or troubleshoot services on this cluster, this document tells you everything you need to know.

**Golden rule:** All manifests go in `~/Desktop/important/talos-cluster/services/manifests/`. Push to Forgejo and Flux auto-deploys within 60 seconds.

---

## Cluster Overview

| Component | Detail |
|-----------|--------|
| Distribution | Talos OS v1.12.5 / Kubernetes v1.35.2 |
| Control Planes | 3× (10.0.0.211, .212, .213) — 4GB RAM each |
| Workers | 3× (10.0.0.214, .215, .216) — 8GB RAM each |
| API VIP | 10.0.0.210 |
| Load Balancer | MetalLB L2 mode, IP pool 10.0.0.220–250 |
| Ingress | Traefik v3 on 10.0.0.220, HTTPS on 443 |
| DNS | ExternalDNS → Pi-hole v6 at 10.0.0.4 |
| TLS | Wildcard cert `*.oryxserver.org` via cert-manager + Cloudflare DNS-01 |
| Storage | NFS on TrueNAS (10.0.0.6), StorageClass `nfs`, path `/mnt/vault/nfs-pvc-kubernetes` |
| Secrets | Sealed Secrets (controller in kube-system) |
| Security | CrowdSec bouncer plugin on Traefik |
| GitOps | Flux watching Forgejo at `http://forgejo.forgejo.svc.cluster.local:3000/jivsan/talos-cluster-gitops.git` |
| Git Remote | `http://git.oryxserver.org/jivsan/talos-cluster-gitops.git` (push here) |
| Container Registry | Forgejo built-in at `git.oryxserver.org` |
| GPU Server | oryxai at 10.0.0.36 — Tesla P100 16GB, Ollama on port 11434 |

---

## Deploying a New Application — Complete Template

Every app follows the same pattern. Copy this template and customize it.

### Step 1: Create the manifest

Save as `~/Desktop/important/talos-cluster/services/manifests/myapp.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data-pvc
  namespace: myapp
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp/myapp:latest
          ports:
            - containerPort: 8080
          env:
            - name: EXAMPLE_VAR
              value: "example-value"
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: myapp-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: myapp
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: traefik-crowdsec@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
    - host: myapp.oryxserver.org
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 8080
  tls:
    - hosts:
        - myapp.oryxserver.org
      secretName: wildcard-oryxserver-tls
```

### Step 2: Copy the wildcard TLS secret to the new namespace

**This is required for every new namespace.** The wildcard cert lives in `traefik` namespace and must be copied:

```bash
kubectl create namespace myapp
kubectl get secret wildcard-oryxserver-tls -n traefik -o yaml | \
  grep -v "resourceVersion\|uid\|creationTimestamp\|last-applied\|labels" | \
  sed "s/namespace: traefik/namespace: myapp/" | \
  kubectl create -f -
```

### Step 3: If the app needs secrets, seal them

**Never commit plain secrets to git.** Use Sealed Secrets:

```bash
# Create the plain secret (gitignored)
cat > secrets/myapp-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: myapp
type: Opaque
stringData:
  DATABASE_URL: "postgres://user:pass@db:5432/myapp"
  API_KEY: "sk-abc123"
EOF

# Seal it (safe to commit)
kubeseal --format yaml \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  < secrets/myapp-secret.yaml > sealed-myapp-secret.yaml
```

Reference sealed secrets in the deployment:

```yaml
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: myapp-secrets
        key: DATABASE_URL
```

Or load all keys at once:

```yaml
envFrom:
  - secretRef:
      name: myapp-secrets
```

### Step 4: Push to Forgejo

```bash
cd ~/Desktop/important/talos-cluster/services/manifests
git add myapp.yaml sealed-myapp-secret.yaml
git commit -m "Deploy myapp"
git push
```

Flux applies within 60 seconds. Force immediate sync:

```bash
flux reconcile source git flux-system
```

### Step 5: Verify

```bash
kubectl -n myapp get pods
curl -k https://myapp.oryxserver.org
```

---

## Common Patterns

### App with PostgreSQL database

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-db-pvc
  namespace: myapp
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-db
  namespace: myapp
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: myapp-db
  template:
    metadata:
      labels:
        app: myapp-db
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: myapp
            - name: POSTGRES_USER
              value: myapp
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: DB_PASSWORD
          volumeMounts:
            - name: pgdata
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: pgdata
          persistentVolumeClaim:
            claimName: myapp-db-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-db
  namespace: myapp
spec:
  ports:
    - port: 5432
  selector:
    app: myapp-db
```

The app connects to the database via `myapp-db:5432` (service DNS within the namespace).

### App with HTTPS backend (self-signed cert)

For apps that serve their own HTTPS (like Crafty, Proxmox, pfSense), use an IngressRoute with ServersTransport:

```yaml
apiVersion: traefik.io/v1alpha1
kind: ServersTransport
metadata:
  name: myapp-transport
  namespace: myapp
spec:
  insecureSkipVerify: true
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: myapp
  namespace: myapp
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`myapp.oryxserver.org`)
      kind: Rule
      middlewares:
        - name: crowdsec
          namespace: traefik
      services:
        - name: myapp
          port: 8443
          scheme: https
          serversTransport: myapp-myapp-transport@kubernetescrd
  tls:
    secretName: wildcard-oryxserver-tls
```

**CRITICAL:** The ServersTransport reference format in Traefik v3 is `namespace-name@kubernetescrd`. For example, a ServersTransport called `myapp-transport` in namespace `myapp` is referenced as `myapp-myapp-transport@kubernetescrd`.

### External service proxy (service outside the cluster)

To proxy an external service through Traefik with wildcard TLS:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-app
  namespace: external
spec:
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-app
  namespace: external
subsets:
  - addresses:
      - ip: 10.0.0.XX
    ports:
      - port: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: external-app
  namespace: external
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: traefik-crowdsec@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
    - host: external-app.oryxserver.org
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: external-app
                port:
                  number: 80
  tls:
    - hosts:
        - external-app.oryxserver.org
      secretName: wildcard-oryxserver-tls
```

**IMPORTANT:** The Service name and Endpoints name MUST match exactly.

### App needing privileged access (e.g., Docker-in-Docker)

Add pod security labels to the namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/warn: privileged
    pod-security.kubernetes.io/audit: privileged
```

### Stateless app (no storage needed)

Simply remove the PVC and volumeMounts from the template. Example:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp/myapp:latest
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: myapp
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: traefik-crowdsec@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
    - host: myapp.oryxserver.org
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 8080
  tls:
    - hosts:
        - myapp.oryxserver.org
      secretName: wildcard-oryxserver-tls
```

### App needing a LoadBalancer IP (non-HTTP, e.g., game server)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-tcp
  namespace: myapp
spec:
  type: LoadBalancer
  ports:
    - port: 25565
      targetPort: 25565
      name: game
  selector:
    app: myapp
```

MetalLB assigns an IP from the pool (10.0.0.220–250).

---

## Critical Rules & Gotchas

### TLS Secret Distribution
The wildcard certificate `wildcard-oryxserver-tls` lives in the `traefik` namespace. **Every namespace that uses it in an Ingress needs its own copy.** This is the #1 cause of 404 errors. Always copy it when creating a new namespace.

### CrowdSec Middleware
Add CrowdSec protection to every Ingress:
- **Standard Ingress:** annotation `traefik.ingress.kubernetes.io/router.middlewares: traefik-crowdsec@kubernetescrd`
- **IngressRoute:** add middleware `{name: crowdsec, namespace: traefik}` to the route

### ExternalDNS
- ExternalDNS watches **standard Ingress objects only** — NOT IngressRoutes
- DNS records for IngressRoutes must be added manually in Pi-hole
- ExternalDNS policy is `upsert-only` — it creates/updates but never deletes

### Kubernetes Service Name Conflicts
If your app name matches a Kubernetes service name, Kubernetes auto-creates environment variables like `MYAPP_PORT=tcp://10.x.x.x:8080`. This can break apps that expect `MYAPP_PORT` to be a number. Fix: explicitly set the conflicting env var (e.g., `JUPYTER_PORT=8888`).

### Image Registries
- Docker Hub: works (default)
- ghcr.io: works
- codeberg.org: **BLOCKED** by cluster network — use `data.forgejo.org` mirror instead
- Forgejo registry: `git.oryxserver.org/owner/repo:tag` (needs imagePullSecret)

### NFS Storage
- StorageClass `nfs` auto-provisions PVCs on TrueNAS
- For existing data, use manual PV/PVC with `storageClassName: ""` pointing to specific NFS path
- TrueNAS NFS share: `10.0.0.6:/mnt/vault/nfs-pvc-kubernetes`

### Flux Sync
- Flux watches the `services/manifests/` directory
- Only Kubernetes manifests (YAML) should be in the root — no Helm values, docs, or scripts
- Helm values go in `helm-values/` (gitignored)
- Plain secrets go in `secrets/` (gitignored)
- **Flux runs with `prune: false`** — deleting a manifest from git does NOT delete the resource from the cluster. Delete namespaces manually.

### Talos-Specific
- No SSH access to nodes — use `talosctl` for node management
- Set `export TALOSCONFIG=~/Desktop/important/talos-cluster/talosconfig` before using talosctl
- Control plane RAM was upgraded to 4GB to prevent OOM — don't reduce it
- Node interface name is `ens18` (not eth0)

---

## Quick Commands

```bash
# Deploy workflow
cd ~/Desktop/important/talos-cluster/services/manifests
git add myapp.yaml sealed-myapp-secret.yaml
git commit -m "Deploy myapp"
git push

# Force Flux sync
flux reconcile source git flux-system
flux reconcile kustomization flux-system

# Check Flux status
flux get sources git
flux get kustomizations

# Copy TLS secret to new namespace
kubectl create namespace myapp
kubectl get secret wildcard-oryxserver-tls -n traefik -o yaml | \
  grep -v "resourceVersion\|uid\|creationTimestamp\|last-applied\|labels" | \
  sed "s/namespace: traefik/namespace: myapp/" | \
  kubectl create -f -

# Seal a secret
kubeseal --format yaml \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  < secrets/plain.yaml > sealed-plain.yaml

# Check all pods
kubectl get pods -A | grep -v Running | grep -v Completed

# Check Traefik logs for routing errors
kubectl -n traefik logs -l app.kubernetes.io/name=traefik --tail=20

# Check CrowdSec bans
kubectl -n crowdsec exec -it $(kubectl -n crowdsec get pod -l app.kubernetes.io/name=crowdsec-lapi -o jsonpath='{.items[0].metadata.name}') -- cscli decisions list

# Restart a deployment
kubectl -n myapp rollout restart deployment myapp

# Delete an app completely
kubectl delete namespace myapp
git rm myapp.yaml sealed-myapp-secret.yaml
git commit -m "Remove myapp" && git push
```

---

## Existing Services Reference

| Service | Namespace | URL | Port |
|---------|-----------|-----|------|
| Rancher | cattle-system | rancher.oryxserver.org | 80 |
| Forgejo | forgejo | git.oryxserver.org | 3000 |
| Crafty | crafty | crafty.oryxserver.org | 8443 (HTTPS) |
| Nextcloud | nextcloud | nextcloud.oryxserver.org | 80 |
| Immich | immich | photos.oryxserver.org | 2283 |
| Jellyfin | jellyfin | jellyfin.oryxserver.org | 8096 |
| OpenWebUI | openwebui | openwebui.oryxserver.org | 8080 |
| SearXNG | searxng | search.oryxserver.org | 8080 |
| Code Server | devtools | code.oryxserver.org | 8080 |
| JupyterLab | devtools | jupyter.oryxserver.org | 8888 |
| IT Tools | it-tools | tools.oryxserver.org | 80 |
| Homepage | homepage | home.oryxserver.org | 3000 |
| TrueNAS (proxy) | external | truenas.oryxserver.org | 80 |
| pfSense (proxy) | external | pfsense.oryxserver.org | 443 |
| Pi-hole (proxy) | external | pihole.oryxserver.org | 80 |
| Grafana (proxy) | external | grafana.oryxserver.org | 3000 |
| Discord Bot | discordbot | N/A (no web UI) | N/A |

---

## Network Map

```
Internet → pfSense (10.0.0.1) → LAN
  ├── Pi-hole DNS (10.0.0.4)
  ├── Proxmox Thor (10.0.0.2)
  ├── Proxmox Hella (10.0.0.3)
  ├── TrueNAS (10.0.0.6) — NFS storage
  ├── Docker VM (10.0.0.15)
  ├── Docker VM (10.0.0.18)
  ├── Monitoring VM (10.0.0.16) — Prometheus + Grafana
  ├── GPU Server oryxai (10.0.0.36) — Tesla P100, Ollama
  ├── Talos CP 1-3 (10.0.0.211-213)
  ├── Talos Workers 1-3 (10.0.0.214-216)
  └── Traefik LB (10.0.0.220) — All *.oryxserver.org traffic
```

# ORYXSERVER Kubernetes Deployment Guide

## Talos OS Cluster — Complete Operations Manual

**Author:** Christina Solbakken  
**Cluster:** talos-cluster (Talos v1.12.5 / Kubernetes v1.35.2)  
**Last Updated:** March 2026

---

## Table of Contents

1. [Cluster Architecture](#1-cluster-architecture)
2. [How Services Work in Kubernetes](#2-how-services-work-in-kubernetes)
3. [Deploying a New Application — Step by Step](#3-deploying-a-new-application--step-by-step)
4. [Managing Secrets with Sealed Secrets](#4-managing-secrets-with-sealed-secrets)
5. [Exposing Services with Ingress & TLS](#5-exposing-services-with-ingress--tls)
6. [Persistent Storage with NFS](#6-persistent-storage-with-nfs)
7. [DNS Automation with ExternalDNS](#7-dns-automation-with-externaldns)
8. [GitOps Workflow with ArgoCD](#8-gitops-workflow-with-argocd)
9. [Practical Examples](#9-practical-examples)
10. [Troubleshooting](#10-troubleshooting)
11. [Quick Reference](#11-quick-reference)

---

## 1. Cluster Architecture

### Nodes

| Node | IP | Role | Host | Resources |
|------|-----|------|------|-----------|
| talos-cp-1 | 10.0.0.211 | Control Plane | thor | 2 vCPU / 2GB RAM |
| talos-cp-2 | 10.0.0.212 | Control Plane | hella | 2 vCPU / 2GB RAM |
| talos-cp-3 | 10.0.0.213 | Control Plane | hella | 2 vCPU / 2GB RAM |
| talos-w-1 | 10.0.0.214 | Worker | thor | 4 vCPU / 8GB RAM |
| talos-w-2 | 10.0.0.215 | Worker | thor | 4 vCPU / 8GB RAM |
| talos-w-3 | 10.0.0.216 | Worker | hella | 4 vCPU / 8GB RAM |

### Key IPs

| Service | IP | Purpose |
|---------|-----|---------|
| Kubernetes API VIP | 10.0.0.210 | Shared control plane endpoint |
| Traefik Ingress | 10.0.0.220 | All HTTP/HTTPS traffic enters here |
| MetalLB Pool | 10.0.0.220–250 | LoadBalancer IP range |

### Infrastructure Stack

Every service you deploy benefits from these components working together:

- **MetalLB** assigns real LAN IPs to LoadBalancer services (layer 2 ARP)
- **Traefik** receives all web traffic on 10.0.0.220 and routes it based on hostname
- **cert-manager** automatically provisions and renews a wildcard TLS certificate for `*.oryxserver.org` via Let's Encrypt + Cloudflare DNS challenge
- **ExternalDNS** watches for new Ingress objects and automatically creates DNS records in Pi-hole pointing to Traefik's IP
- **NFS Provisioner** dynamically creates persistent storage directories on TrueNAS when pods request it
- **Sealed Secrets** encrypts secrets so they can safely live in your Git repo
- **ArgoCD** watches your Git repo and auto-applies changes to the cluster

### Why This Architecture?

The goal is **zero-touch deployment**. When you create a new service manifest and push it to Git:

1. ArgoCD detects the change and applies it
2. Kubernetes schedules the pods on worker nodes
3. MetalLB assigns IPs if a LoadBalancer service is requested
4. Traefik picks up the Ingress and starts routing traffic
5. cert-manager provides the wildcard TLS cert automatically
6. ExternalDNS creates the DNS record in Pi-hole
7. The service is accessible at `https://yourapp.oryxserver.org` within ~60 seconds

No manual DNS entries. No manual certificate configuration. No manual deployment commands.

---

## 2. How Services Work in Kubernetes

Every application in Kubernetes is made up of several resource types. Understanding why each exists is critical.

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
```

**What it does:** Creates an isolated environment for your app. All resources (pods, services, secrets) live inside this namespace.

**Why:** Keeps things organized. You can delete an entire app by deleting its namespace. It also prevents name collisions — two apps can both have a service called "redis" if they're in different namespaces.

### Deployment

```yaml
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
          image: myimage:latest
          ports:
            - containerPort: 8080
```

**What it does:** Tells Kubernetes to run your container image and keep it running. If the pod crashes, Kubernetes restarts it. If a worker node dies, Kubernetes reschedules the pod on another node.

**Why each field matters:**

- `replicas: 1` — How many copies of your app to run. Use 1 for stateful apps (databases, apps with persistent storage). Use 2+ for stateless apps you want highly available.
- `strategy: Recreate` — How to handle updates. `Recreate` stops the old pod before starting the new one (required when only one pod can access a volume at a time). `RollingUpdate` (default) starts new pods before stopping old ones for zero-downtime deploys.
- `selector.matchLabels` — Must match `template.metadata.labels`. This is how the Deployment knows which pods belong to it.
- `containerPort` — Declares what port the app listens on inside the container. This is informational but good practice.

### Service

```yaml
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
```

**What it does:** Creates a stable internal DNS name and IP for your app. Other pods in the cluster can reach it at `myapp.myapp.svc.cluster.local:8080`.

**Why:** Pod IPs change every time they restart. A Service gives you a stable endpoint. The `selector` matches pods with the label `app: myapp`.

**Service types:**

- `ClusterIP` (default) — Only accessible inside the cluster. Perfect for internal services like databases, Redis, or apps behind an Ingress.
- `LoadBalancer` — MetalLB assigns a real LAN IP. Use for services that need direct access from your network (e.g., Minecraft server on port 25565, game servers, MQTT).

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
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

**What it does:** Tells Traefik to route requests for `myapp.oryxserver.org` to your Service. Also triggers ExternalDNS to create the Pi-hole record automatically.

**Why each field matters:**

- `ingressClassName: traefik` — Required. Tells Kubernetes which ingress controller handles this.
- `host` — The hostname. Must be a subdomain of `oryxserver.org` to use the wildcard cert.
- `backend.service` — Points to the Service name and port defined above.
- `tls.secretName: wildcard-oryxserver-tls` — References the wildcard certificate managed by cert-manager. You never need to create or manage this — it's shared across all Ingress objects in the cluster via the TLSStore.

**When to use Ingress vs LoadBalancer:**

- **Ingress** — For HTTP/HTTPS web services (web UIs, APIs, dashboards). All share Traefik's IP (10.0.0.220) and are differentiated by hostname.
- **LoadBalancer** — For non-HTTP protocols (Minecraft TCP, game UDP, MQTT, raw TCP/UDP). Each gets its own unique IP from MetalLB.
- **Both** — Some apps need both. Crafty uses an Ingress for the web panel and a LoadBalancer for the Minecraft port.

---

## 3. Deploying a New Application — Step by Step

### The Complete Workflow

Here is the full process for deploying any new application from scratch.

#### Step 1: Create the namespace and manifest

```bash
cd ~/Desktop/important/talos-cluster/services/manifests
```

Create a YAML file named `myapp.yaml`:

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
          image: someimage:latest
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

#### Step 2: Deploy manually (for testing) or via GitOps

**Manual (for testing):**

```bash
kubectl apply -f myapp.yaml
kubectl -n myapp get pods
```

**GitOps (production — preferred):**

```bash
git add myapp.yaml
git commit -m "Add myapp"
git push
```

ArgoCD will detect the change within 3 minutes and deploy automatically.

#### Step 3: Verify

```bash
# Check pod is running
kubectl -n myapp get pods

# Check service exists
kubectl -n myapp get svc

# Check ingress is created
kubectl -n myapp get ingress

# Check DNS was created (wait ~60 seconds)
nslookup myapp.oryxserver.org 10.0.0.4

# Check logs if something is wrong
kubectl -n myapp logs -l app=myapp --tail=30
```

#### Step 4: Access

Open `https://myapp.oryxserver.org` in your browser. TLS is automatic.

---

## 4. Managing Secrets with Sealed Secrets

### Why Sealed Secrets?

Your Git repo should contain everything needed to recreate the cluster. But secrets (passwords, API tokens, keys) can't be stored in plain text — even in a private repo. Sealed Secrets solves this by encrypting secrets with a key that only your cluster has.

### How It Works

1. You write a plain Kubernetes Secret YAML (never commit this)
2. `kubeseal` encrypts it using the cluster's public key
3. The encrypted `SealedSecret` YAML is safe to commit
4. The sealed-secrets-controller in the cluster decrypts it back into a regular Secret

### Creating a Sealed Secret

#### Step 1: Write the plain secret (in the gitignored `secrets/` directory)

```bash
cat > secrets/myapp-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: myapp
type: Opaque
stringData:
  DATABASE_URL: "postgresql://user:password@host:5432/db"
  API_KEY: "super-secret-key"
EOF
```

**Important:** This file is in `secrets/` which is in `.gitignore`. It never gets committed.

#### Step 2: Seal it

```bash
kubeseal --format yaml \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  < secrets/myapp-secret.yaml > sealed-myapp-secret.yaml
```

#### Step 3: Commit the sealed version

```bash
git add sealed-myapp-secret.yaml
git commit -m "Add myapp sealed secret"
git push
```

#### Step 4: Reference the secret in your Deployment

The sealed secret gets decrypted into a regular Kubernetes Secret with the same name. Reference it with `envFrom` or individual `env` entries:

**All variables at once:**

```yaml
containers:
  - name: myapp
    image: myimage:latest
    envFrom:
      - secretRef:
          name: myapp-secrets
```

**Individual variables:**

```yaml
containers:
  - name: myapp
    image: myimage:latest
    env:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: myapp-secrets
            key: DATABASE_URL
```

### Updating a Secret

1. Edit the plain secret in `secrets/`
2. Re-seal it (same command as Step 2)
3. Commit and push the updated sealed file
4. Restart the deployment to pick up the new values:

```bash
kubectl -n myapp rollout restart deployment myapp
```

### Important Notes

- Sealed Secrets are bound to the **namespace and name** in the metadata. You can't change these after sealing.
- If you rebuild the cluster from scratch, you need to restore the sealed-secrets controller key pair, or re-seal all secrets.
- Back up the controller key: `kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-key-backup.yaml`

---

## 5. Exposing Services with Ingress & TLS

### Standard HTTP/HTTPS Service (Most Common)

For any web application, use an Ingress. Traefik handles TLS termination, and the wildcard cert covers all `*.oryxserver.org` subdomains.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
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

### Backend Serves HTTPS (e.g., Crafty, ArgoCD)

Some apps serve their own HTTPS with self-signed certs. Traefik needs to talk HTTPS to the backend and skip cert verification. Use a Traefik IngressRoute instead:

```yaml
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
      services:
        - name: myapp
          port: 8443
          scheme: https
          serversTransport: myapp-transport
  tls:
    secretName: wildcard-oryxserver-tls
---
apiVersion: traefik.io/v1alpha1
kind: ServersTransport
metadata:
  name: myapp-transport
  namespace: myapp
spec:
  insecureSkipVerify: true
```

**Important:** ExternalDNS does NOT watch IngressRoutes — only standard Ingress objects. If you use IngressRoutes, you must manually create the DNS record in Pi-hole, or create a dummy Ingress alongside it for DNS purposes.

### Non-HTTP Service (TCP/UDP — e.g., Minecraft, game servers)

For raw TCP/UDP, use a LoadBalancer service. MetalLB assigns a unique IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-tcp
  namespace: myapp
  annotations:
    external-dns.alpha.kubernetes.io/hostname: myapp.oryxserver.org
spec:
  type: LoadBalancer
  ports:
    - port: 25565
      targetPort: 25565
      protocol: TCP
  selector:
    app: myapp
```

The `external-dns.alpha.kubernetes.io/hostname` annotation tells ExternalDNS to create a DNS record pointing to the LoadBalancer IP (not Traefik).

### Combined: Web UI + Game Port

For apps like Crafty (web panel + Minecraft server), use both:

```yaml
# ClusterIP for web (behind Ingress)
apiVersion: v1
kind: Service
metadata:
  name: myapp-web
  namespace: myapp
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: myapp
---
# LoadBalancer for game port
apiVersion: v1
kind: Service
metadata:
  name: myapp-game
  namespace: myapp
  annotations:
    external-dns.alpha.kubernetes.io/hostname: game.oryxserver.org
spec:
  type: LoadBalancer
  ports:
    - port: 25565
      targetPort: 25565
  selector:
    app: myapp
---
# Ingress for web
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
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
                name: myapp-web
                port:
                  number: 8080
  tls:
    - hosts:
        - myapp.oryxserver.org
      secretName: wildcard-oryxserver-tls
```

---

## 6. Persistent Storage with NFS

### Why Persistent Storage?

By default, everything inside a container is ephemeral — if the pod restarts, all data is lost. Persistent storage keeps data on TrueNAS so it survives restarts, rescheduling, and even cluster rebuilds.

### Option A: Dynamic Provisioning (Simple — No Data to Migrate)

For fresh installs where you don't care about the exact NFS path, use a PersistentVolumeClaim with the default storage class. The NFS provisioner auto-creates a subdirectory on TrueNAS.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-pvc
  namespace: myapp
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Reference it in the Deployment:

```yaml
spec:
  containers:
    - name: myapp
      volumeMounts:
        - name: data
          mountPath: /app/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: myapp-pvc
```

The provisioner creates: `/mnt/vault/nfs-pvc-kubernetes/<namespace>-<pvc-name>-<random>/`

### Option B: Static PV Pointing to Existing Data

When migrating from Docker or when you need a specific TrueNAS path (e.g., your media library, Nextcloud data), create a PersistentVolume pointing to the exact NFS path.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: myapp-data-pv
spec:
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 10.0.0.6
    path: /mnt/vault/some-existing-dataset
  persistentVolumeReclaimPolicy: Retain
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
      storage: 500Gi
  volumeName: myapp-data-pv
  storageClassName: ""
```

**Important fields:**

- `storageClassName: ""` — Required on the PVC to prevent dynamic provisioning. Without this, Kubernetes ignores the `volumeName` and creates a new volume.
- `volumeName: myapp-data-pv` — Binds the PVC to the specific PV.
- `persistentVolumeReclaimPolicy: Retain` — Keeps the data on TrueNAS even if you delete the PV. Never use `Delete` for data you care about.
- `accessModes: ReadWriteOnce` — One pod can write. Use `ReadOnlyMany` for shared read-only data (like a media library mounted in multiple pods).

### Multiple Volumes with subPath

For apps that need multiple directories (like Crafty with backups, servers, config), use `subPath` to mount different subdirectories from the same PVC:

```yaml
volumeMounts:
  - name: data
    mountPath: /app/config
    subPath: config
  - name: data
    mountPath: /app/data
    subPath: data
  - name: data
    mountPath: /app/backups
    subPath: backups
```

---

## 7. DNS Automation with ExternalDNS

### How It Works

ExternalDNS runs as a pod in the cluster and watches for two things:

1. **Ingress objects** — Creates an A record for the `host` field, pointing to Traefik's IP (10.0.0.220)
2. **LoadBalancer Services** with the annotation `external-dns.alpha.kubernetes.io/hostname` — Creates an A record pointing to the MetalLB-assigned IP

Records are created in Pi-hole via its v6 REST API.

### What Happens Automatically

When you create this Ingress:

```yaml
spec:
  rules:
    - host: myapp.oryxserver.org
```

Within ~60 seconds, ExternalDNS creates: `myapp.oryxserver.org → 10.0.0.220` in Pi-hole.

### What Does NOT Happen Automatically

- **IngressRoute** (Traefik CRD) — ExternalDNS doesn't watch these. Use standard Ingress for DNS automation.
- **Wildcard records** — Pi-hole doesn't support wildcards via ExternalDNS.
- **Record deletion** — Our ExternalDNS runs with `--policy=upsert-only`, meaning it creates and updates records but never deletes them. This protects manually-created records. Clean up stale records manually in Pi-hole.

### Verifying DNS

```bash
# Check what ExternalDNS is doing
kubectl -n external-dns logs -l app=external-dns --tail=20

# Verify a record exists
nslookup myapp.oryxserver.org 10.0.0.4
```

---

## 8. GitOps Workflow with ArgoCD

### The Workflow

Your Git repo at `github.com/jivsan/talos-cluster-gitops` is the single source of truth. ArgoCD continuously compares the repo state to the cluster state.

#### To deploy a new app:

```bash
cd ~/Desktop/important/talos-cluster/services/manifests

# Create your manifest
nano myapp.yaml

# If the app needs secrets, seal them
kubeseal --format yaml \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  < secrets/myapp-secret.yaml > sealed-myapp-secret.yaml

# Commit and push
git add myapp.yaml sealed-myapp-secret.yaml
git commit -m "Deploy myapp"
git push
```

ArgoCD detects the change within ~3 minutes and applies it.

#### To update an app:

Edit the manifest, commit, push. ArgoCD applies the diff.

```bash
# Example: update image version
sed -i 's|image: myapp:v1|image: myapp:v2|' myapp.yaml
git add myapp.yaml
git commit -m "Update myapp to v2"
git push
```

#### To delete an app:

Remove the manifest from git. With auto-sync and self-heal enabled, ArgoCD will remove the resources from the cluster.

```bash
git rm myapp.yaml
git commit -m "Remove myapp"
git push
```

### Monitoring Sync Status

- **ArgoCD UI:** https://argocd.oryxserver.org — Visual tree of all resources, sync status, health
- **CLI:** `argocd app list` (if argocd CLI is installed)

### Manual Sync

If you don't want to wait for auto-sync, click **Sync** in the ArgoCD UI or run:

```bash
argocd app sync talos-cluster --grpc-web
```

---

## 9. Practical Examples

### Example 1: Stateless Web App (Simplest)

A tool like IT-Tools — no database, no persistent storage, just a web UI.

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
          image: someimage:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: myapp
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
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
                  number: 80
  tls:
    - hosts:
        - myapp.oryxserver.org
      secretName: wildcard-oryxserver-tls
```

### Example 2: App with Database and Secrets

A typical app with PostgreSQL, secrets, and persistent storage.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
# Database
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
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: DB_USER
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
---
# Application
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
      storage: 20Gi
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
          image: myapp:latest
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: DATABASE_URL
          volumeMounts:
            - name: data
              mountPath: /app/data
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

Create the sealed secret:

```bash
cat > secrets/myapp-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: myapp
type: Opaque
stringData:
  DB_USER: "myapp"
  DB_PASSWORD: "super-secure-password"
  DATABASE_URL: "postgresql://myapp:super-secure-password@myapp-db:5432/myapp"
EOF

kubeseal --format yaml \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  < secrets/myapp-secret.yaml > sealed-myapp-secret.yaml
```

### Example 3: Docker Compose Migration

Converting a Docker Compose file to Kubernetes. Take this compose:

```yaml
services:
  app:
    image: myapp:latest
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=db
      - REDIS_HOST=redis
    volumes:
      - ./data:/app/data
    depends_on:
      - db
      - redis
  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - db-data:/var/lib/postgresql/data
  redis:
    image: redis:alpine
```

The translation rules are:

| Docker Compose | Kubernetes |
|---------------|------------|
| `services.app` | Deployment + Service |
| `ports: "3000:3000"` | Service port + Ingress or LoadBalancer |
| `environment` | `env` in container spec, or Secret + `envFrom` |
| `volumes: ./data:/app/data` | PVC + volumeMount |
| `depends_on` | Not needed — k8s handles restarts. App should handle connection retries. |
| `networks` | Not needed — all pods in a namespace can reach each other via Service names |
| Service name `db` | Service named `db` in same namespace = same DNS name |

---

## 10. Troubleshooting

### Pod won't start

```bash
# Check pod status
kubectl -n myapp get pods

# Describe the pod for events
kubectl -n myapp describe pod <pod-name>

# Check logs
kubectl -n myapp logs <pod-name> --tail=50

# If the pod keeps restarting, check previous logs
kubectl -n myapp logs <pod-name> --previous --tail=50
```

### Common pod states and fixes

| Status | Cause | Fix |
|--------|-------|-----|
| `ImagePullBackOff` | Can't download the image | Check image name, or add imagePullSecrets for private registries |
| `CrashLoopBackOff` | App starts and immediately crashes | Check logs: `kubectl logs <pod>` |
| `Pending` | No node has enough resources, or PVC not bound | Check events: `kubectl describe pod <pod>` |
| `ContainerCreating` | Pulling image or mounting volumes | Wait, or check events for mount failures |
| `CreateContainerConfigError` | Missing secret or configmap | Check the secret/configmap exists in the same namespace |

### Service not reachable

```bash
# Is the service resolving?
nslookup myapp.oryxserver.org 10.0.0.4

# Does the ingress exist?
kubectl -n myapp get ingress

# Is Traefik routing it?
curl -k -H "Host: myapp.oryxserver.org" https://10.0.0.220

# Can you reach the pod directly?
kubectl -n myapp port-forward svc/myapp 8888:8080
# Then test http://localhost:8888
```

### DNS not updating

```bash
# Check ExternalDNS logs
kubectl -n external-dns logs -l app=external-dns --tail=20

# ExternalDNS only watches Ingress objects, NOT IngressRoutes
# Make sure you're using a standard Ingress
```

### NFS mount failed

```bash
# Check TrueNAS is reachable
ping 10.0.0.6

# Test NFS mount manually
sudo mount -t nfs 10.0.0.6:/mnt/vault/nfs-pvc-kubernetes /tmp/nfstest
ls /tmp/nfstest
sudo umount /tmp/nfstest

# Check the PV path exists on TrueNAS
# The NFS path in the PV must exactly match the TrueNAS dataset path
```

### API server unreachable (connection refused to 10.0.0.210)

The VIP occasionally hiccups. Usually resolves itself within seconds. Just retry the command. If persistent:

```bash
# Check control plane health
talosctl -e 10.0.0.211 -n 10.0.0.211 health

# Check etcd
talosctl -e 10.0.0.211 -n 10.0.0.211 etcd members
```

---

## 11. Quick Reference

### Essential kubectl commands

```bash
# See all pods across all namespaces
kubectl get pods -A

# See pods in a specific namespace
kubectl -n myapp get pods

# See everything in a namespace
kubectl -n myapp get all

# View pod logs (live follow)
kubectl -n myapp logs -l app=myapp -f

# Exec into a running container
kubectl -n myapp exec -it <pod-name> -- /bin/sh

# Restart a deployment (rolling restart)
kubectl -n myapp rollout restart deployment myapp

# Scale replicas
kubectl -n myapp scale deployment myapp --replicas=3

# Delete a namespace and everything in it
kubectl delete namespace myapp

# Set environment variable on a deployment
kubectl -n myapp set env deployment/myapp MY_VAR=value

# Check resource usage
kubectl top nodes
kubectl top pods -A
```

### talosctl commands

```bash
# Check cluster health
talosctl -e 10.0.0.211 -n 10.0.0.211 health

# Check etcd members
talosctl -e 10.0.0.211 -n 10.0.0.211 etcd members

# View services on a node
talosctl -e 10.0.0.211 -n 10.0.0.211 services

# Get kubeconfig
talosctl -e 10.0.0.211 -n 10.0.0.211 kubeconfig ~/.kube/config

# View node logs
talosctl -e 10.0.0.211 -n 10.0.0.211 logs kubelet

# Check extensions installed
talosctl -e 10.0.0.211 -n 10.0.0.211 get extensions
```

### Sealed Secrets commands

```bash
# Seal a secret
kubeseal --format yaml \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  < secrets/plain-secret.yaml > sealed-secret.yaml

# Fetch the public cert (for offline sealing)
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system > sealed-secrets-cert.pem

# Backup the controller key (IMPORTANT — store securely)
kubectl -n kube-system get secret \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-key-backup.yaml
```

### File locations

| What | Path |
|------|------|
| Manifests | `~/Desktop/important/talos-cluster/services/manifests/` |
| Plain secrets (gitignored) | `~/Desktop/important/talos-cluster/services/manifests/secrets/` |
| Talos configs | `~/Desktop/important/talos-cluster/` |
| talosconfig | `~/Desktop/important/talos-cluster/talosconfig` |
| kubeconfig | `~/.kube/config` |
| Git repo | `github.com/jivsan/talos-cluster-gitops` |

### Service URLs

| Service | URL |
|---------|-----|
| Rancher | https://rancher.oryxserver.org |
| ArgoCD | https://argocd.oryxserver.org |
| Crafty | https://crafty.oryxserver.org |
| Nextcloud | https://nextcloud.oryxserver.org |
| Immich | https://photos.oryxserver.org |
| Jellyfin | https://jellyfin.oryxserver.org |
| SearXNG | https://search.oryxserver.org |
| Code Server | https://code.oryxserver.org |
| JupyterLab | https://jupyter.oryxserver.org |
| IT Tools | https://tools.oryxserver.org |
| Homepage | https://home.oryxserver.org |
| Grafana | http://10.0.0.16:3000 |
| Pi-hole | http://10.0.0.4/admin |

---

## Appendix: Backup Strategy

### What to back up

1. **Sealed Secrets controller key** — Without this, you can't decrypt your sealed secrets after a cluster rebuild
2. **Talos machine configs** — `cp1.yaml`, `cp2.yaml`, etc. in your talos-cluster directory
3. **Git repo** — Contains all manifests. GitHub is the backup.
4. **TrueNAS snapshots** — Your NFS data is the most critical. Set up automatic ZFS snapshots.
5. **etcd** — Talos can snapshot etcd: `talosctl -e 10.0.0.211 -n 10.0.0.211 etcd snapshot /tmp/etcd-backup.db`

### What you DON'T need to back up

- Pod state / container images — Everything is declarative and pulled from registries
- Kubernetes cluster state — Fully reproducible from your Git repo + Talos configs
- Ingress/DNS/TLS — All auto-generated from manifests

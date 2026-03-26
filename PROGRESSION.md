# Homelab k3s Platform — Build Progression

Step-by-step build log for a prod-like k3s cluster on Proxmox.  
Ubuntu nodes, 3-node cluster (1 control plane + 2 workers).

## Stack Overview

| Layer | Component |
|-------|-----------|
| Networking | MetalLB (L2), Istio (service mesh + ingress gateway) |
| Tunnel | Cloudflare Tunnel — zero public IP exposure |
| TLS | cert-manager + Let's Encrypt DNS-01 via Cloudflare API |
| GitOps | ArgoCD (App-of-Apps pattern) |
| Storage | Longhorn (distributed block, 2 replicas) |
| Secrets | Sealed Secrets |
| Observability | Grafana LGTM stack (Loki, Grafana, Tempo, Mimir) + Alloy collector, Kiali |
| CI/CD | Gitea + Gitea Actions |

---

## Phase 1 — Foundation

### Step 1 — Prepare the nodes

> Run on **all 3 nodes**.

Install `open-iscsi` — required by Longhorn to mount block volumes:

```bash
sudo apt install -y open-iscsi
sudo systemctl enable --now iscsid
systemctl status iscsid  # should show: active (running)
```

Disable k3s's built-in Traefik — Istio Ingress Gateway will replace it:

> Run on the **control plane node** only.

```bash
sudo nano /etc/rancher/k3s/config.yaml
```

Add:
```yaml
disable:
  - traefik
```

Restart k3s and verify Traefik is gone:

```bash
sudo systemctl restart k3s
kubectl get pods -n kube-system
# traefik pod should no longer appear
```

---

### Step 2 — Bootstrap ArgoCD

> ArgoCD is the **only component installed manually**.  
> After this, ArgoCD manages everything else via GitOps.

> Run on the **control plane node** (or any machine with `~/.kube/config` configured).

```bash
# Add Argo Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create namespace
kubectl create namespace argocd

# Install ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 9.4.15 \
  --set server.insecure=true \
  --set configs.params."server\.insecure"=true

# Watch pods come up (~1-2 min)
kubectl get pods -n argocd -w
```

> `server.insecure=true` — ArgoCD skips its own TLS. Istio Gateway handles TLS termination. This is intentional.

Get the initial admin password (this works if you are on a linux host):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Access the UI temporarily via port-forward:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
# Open http://localhost:8080 — login: admin / <password above>
```

---

### Step 3 — Install Sealed Secrets

**Why:** Before committing anything to Git, we need a way to store secrets safely. Sealed Secrets encrypts a Kubernetes Secret into a `SealedSecret` — safe to commit to Git. Only the in-cluster controller can decrypt it.

> Run on the **control plane node**.

```bash
# Add Sealed Secrets Helm repo
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

# Install into kube-system
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set fullnameOverride=sealed-secrets-controller

# Verify it's running
kubectl get pods -n kube-system | grep sealed-secrets
```

Install the `kubeseal` CLI (run on **control plane** or **local machine**):

```bash
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest \
  | grep tag_name | cut -d '"' -f 4 | sed 's/v//')

wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz
tar xfz kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
kubeseal --version
```

---

### Step 4 — Install MetalLB

**Why:** k3s doesn't provide a real `LoadBalancer` by default. MetalLB assigns actual IPs from your LAN to `LoadBalancer` services — Istio's Ingress Gateway needs one to get a stable cluster IP.

> Run on the **control plane node**.

```bash
# Add MetalLB Helm repo
helm repo add metallb https://metallb.github.io/metallb
helm repo update

# Install MetalLB
helm install metallb metallb/metallb --namespace metallb-system --create-namespace

# Watch pods come up (~1 min)
kubectl get pods -n metallb-system -w
```

Once all pods are `Running`, configure the IP pool (`10.10.10.240–10.10.10.250`):

```bash
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
    - 10.10.10.240-10.10.10.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
    - default
EOF
```

Verify the pool was created:

```bash
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

---

### Step 5 — Install Istio

**Why:** Istio is the service mesh. It gives us:
- **Ingress Gateway** — replaces Traefik, receives traffic from the Cloudflare tunnel
- **Envoy sidecars** — injected into every pod, handle mTLS between services automatically
- **mTLS STRICT** — all service-to-service traffic is encrypted and authenticated
- **AuthorizationPolicies** — L7-aware access control (replaces NetworkPolicies)

We install Istio in 3 parts: `base` (CRDs), `istiod` (control plane), `gateway` (ingress).

> Run on the **control plane node**.

```bash
# Add Istio Helm repo
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

# 1. Install CRDs and base resources
helm install istio-base istio/base --namespace istio-system --create-namespace --set defaultRevision=default

# 2. Install istiod (control plane)
helm install istiod istio/istiod --namespace istio-system --set meshConfig.accessLogFile=/dev/stdout --set meshConfig.enableTracing=true --wait

# 3. Install the Ingress Gateway
helm install istio-gateway istio/gateway --namespace istio-system

# Watch the gateway get a LoadBalancer IP from MetalLB (~30s)
kubectl get svc -n istio-system -w
```

The gateway service should receive an `EXTERNAL-IP` from the `10.10.10.240-10.10.10.250` pool:
```
istio-gateway   LoadBalancer   ...   10.10.10.240   ...
```

Enable sidecar injection for workload namespaces:

```bash
kubectl create namespace infrastructure
kubectl create namespace monitoring
kubectl create namespace staging
kubectl create namespace production

kubectl label namespace infrastructure istio-injection=enabled
kubectl label namespace monitoring istio-injection=enabled
kubectl label namespace staging istio-injection=enabled
kubectl label namespace production istio-injection=enabled
```

Verify Istio is healthy:

```bash
kubectl get pods -n istio-system
```

---

### Step 6 — cert-manager + Cloudflare DNS-01

**Why:** cert-manager automates TLS certificate issuance. We use Cloudflare's DNS API to solve ACME DNS-01 challenges — this works for wildcard certs (`*.lab.kovachev-projects.com`) and requires no inbound HTTP traffic.

We start with the **staging** issuer to validate the setup without hitting Let's Encrypt production rate limits (max 5 duplicate certs/week). Once staging succeeds, we switch to production.

#### 6a — Seal the Cloudflare API token

> Run from your **local Windows machine**.

```powershell
kubectl create secret generic cloudflare-api-token `
  --namespace cert-manager `
  --from-literal=api-token=<YOUR_CF_API_TOKEN> `
  --dry-run=client -o yaml `
| kubeseal --controller-namespace kube-system -o yaml
```

Save the output to `infrastructure/cert-manager/cloudflare-api-token.sealedsecret.yaml`.

#### 6b — Install cert-manager

> Run on the **control plane node**.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

kubectl get pods -n cert-manager -w
# Wait for: cert-manager, cert-manager-cainjector, cert-manager-webhook → Running
```

#### 6c — Apply ClusterIssuers, SealedSecret and Certificate

> Run from your **local Windows machine**.

```powershell
kubectl apply -f infrastructure\cert-manager\cloudflare-api-token.sealedsecret.yaml
kubectl apply -f infrastructure\cert-manager\clusterissuers.yaml
kubectl apply -f infrastructure\cert-manager\wildcard-cert.yaml
```

Watch the staging cert get issued via Cloudflare DNS-01 challenge (~1-2 min):

```bash
kubectl get certificate -n istio-system -w
# READY should go False → True
```

#### 6d — Switch to production issuer

Once staging shows `READY=True`, update the cert to use the production issuer:

```powershell
# Update the issuer in the manifest
(Get-Content infrastructure\cert-manager\wildcard-cert.yaml) `
  -replace 'letsencrypt-staging', 'letsencrypt-production' `
| Set-Content infrastructure\cert-manager\wildcard-cert.yaml

kubectl apply -f infrastructure\cert-manager\wildcard-cert.yaml

# Watch re-issuance
kubectl get certificate -n istio-system -w
```

Verify:
```bash
kubectl describe certificate homelab-wildcard-cert -n istio-system | grep Issuer
# Should show: letsencrypt-production
```

---

### Step 7 — Deploy Cloudflare Tunnel

**Why:** Instead of exposing your cluster's IP publicly, `cloudflared` makes an **outbound-only** connection to Cloudflare's edge. Traffic flows: `User → Cloudflare Edge → cloudflared pod → Istio Ingress Gateway`. No port forwarding, no firewall rules, no public IP.

We run 2 replicas of cloudflared for HA, spread across both worker nodes.

#### 7a — Create the tunnel in Cloudflare

**Option 1 — Browser UI (recommended)**

1. Go to **Cloudflare Dashboard → Zero Trust → Networks → Tunnels → Create a tunnel**
2. Choose **Cloudflared** as the connector type
3. Name it `serverden-homelab` → click Save
4. Copy the **tunnel token** shown on the next screen (long string starting with `eyJ...`)
5. Skip the "Install connector" step — we deploy it in k8s ourselves
6. Finish the wizard — public hostnames are configured later

> The browser UI gives you a single token string. This is simpler than the CLI approach — no credentials JSON file needed.

**Option 2 — CLI (alternative)**

> Run on the control plane node or any machine with `cloudflared` installed.

```bash
# Install cloudflared
curl -L https://pkg.cloudflare.com/cloudflare-main.gpg | sudo gpg --dearmor -o /usr/share/keyrings/cloudflare-main.gpg
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared

# Authenticate (opens browser) and create the tunnel
cloudflared tunnel login
cloudflared tunnel create serverden-homelab

# Seal the credentials JSON file
kubectl create secret generic cloudflared-credentials \
  --namespace infrastructure \
  --from-file=credentials.json=$HOME/.cloudflared/<TUNNEL_ID>.json \
  --dry-run=client -o yaml \
| kubeseal --controller-namespace kube-system -o yaml \
> infrastructure/cloudflare/tunnel-credentials.sealedsecret.yaml
```

#### 7b — Seal the tunnel token

> Run from your **local Windows machine**.

```powershell
kubectl create secret generic cloudflared-token `
  --namespace infrastructure `
  --from-literal=token=<TUNNEL_TOKEN> `
  --dry-run=client -o yaml `
| kubeseal --controller-namespace kube-system -o yaml
```

Save the output to `infrastructure/cloudflare/cloudflared-token.sealedsecret.yaml`.

#### 7c — Create the cloudflared Deployment

Create `infrastructure/cloudflare/cloudflared-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: infrastructure
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:2026.3.0
          args:
            - tunnel
            - --no-autoupdate
            - --metrics
            - 0.0.0.0:2000
            - run
          env:
            - name: TUNNEL_TOKEN
              valueFrom:
                secretKeyRef:
                  name: cloudflared-token
                  key: token
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          livenessProbe:
            httpGet:
              path: /ready
              port: 2000
            initialDelaySeconds: 10
            periodSeconds: 10
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: cloudflared
```

#### 7d — Apply the manifests

```powershell
kubectl apply -f infrastructure\cloudflare\cloudflared-token.sealedsecret.yaml
kubectl apply -f infrastructure\cloudflare\cloudflared-deployment.yaml

# Watch pods come up
kubectl get pods -n infrastructure -w
```

#### 7e — Configure SSL/TLS mode

Set SSL/TLS mode to **Full (strict)**: Cloudflare Dashboard → SSL/TLS → Overview.

> **Domain note:** Cloudflare Universal SSL only covers one level deep (`*.kovachev-projects.com`).  
> A second-level wildcard like `*.lab.kovachev-projects.com` requires Advanced Certificate Manager ($10/mo).  
> All services are therefore exposed directly under `*.kovachev-projects.com` (e.g. `argocd.kovachev-projects.com`).

Public hostnames are added **per service** via the tunnel dashboard (see Step 9). Cloudflare auto-creates the CNAME DNS record for each one.

#### 7f — Verify the tunnel

```bash
kubectl logs -n infrastructure -l app=cloudflared --tail=20
# Should show: "Connection registered" for both replicas
```

Check tunnel status in the Cloudflare dashboard — it should show **Healthy** with 2 connectors.

---

### Step 8 — Deploy Longhorn

**Why:** Longhorn provides distributed block storage across all 3 nodes using the dedicated disk on each. It creates replicated PersistentVolumes — if a node goes down, your data is still accessible from another replica. We set 2 replicas (safe on 2 workers, tolerates 1 failure).

**Pre-requisite:** `open-iscsi` must be running on all nodes (done in Step 1).

> Run on the **control plane node**.

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --version 1.10.2 \
  --set defaultSettings.defaultReplicaCount=2 \
  --set defaultSettings.storageMinimalAvailablePercentage=10

# Watch all pods come up — takes 2-3 min
kubectl get pods -n longhorn-system -w
```

Verify Longhorn is the default StorageClass:

```bash
kubectl get storageclass
# longhorn should show: (default)
```

If it's not marked default:
```bash
kubectl patch storageclass longhorn \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Test with a quick PVC:
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: default
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc test-pvc
# STATUS should show: Bound

# Clean up
kubectl delete pvc test-pvc
```

---

### Step 9 — Expose services via Istio Gateway + Cloudflare Tunnel

This step wires a service through the full public path:  
`User → Cloudflare Edge (TLS) → cloudflared pod → Istio Gateway → VirtualService → Service`

ArgoCD is the first service we expose. Repeat this pattern for every service.

#### 9a — Create the Istio Gateway CRD

Create `infrastructure/istio/gateway.yaml`:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: homelab-gateway
  namespace: istio-system
spec:
  selector:
    istio: gateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: homelab-wildcard-cert
      hosts:
        - "*.kovachev-projects.com"
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*.kovachev-projects.com"
      tls:
        httpsRedirect: true
```

> `httpsRedirect: true` on port 80 ensures any plain HTTP is redirected to HTTPS.  
> The Cloudflare tunnel **must** connect over HTTPS:443 — see the public hostname config below.

Apply it:
```bash
kubectl apply -f infrastructure/istio/gateway.yaml
```

#### 9b — Create a VirtualService for ArgoCD

Create `infrastructure/argocd/virtualservice.yaml`:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: argocd
  namespace: argocd
spec:
  hosts:
    - argocd.kovachev-projects.com
  gateways:
    - istio-system/homelab-gateway
  http:
    - route:
        - destination:
            host: argocd-server.argocd.svc.cluster.local
            port:
              number: 80
```

> ArgoCD is configured with `server.insecure=true` — it serves HTTP. Istio Gateway handles TLS termination.

Apply it:
```bash
kubectl apply -f infrastructure/argocd/virtualservice.yaml
```

#### 9c — Add a public hostname in Cloudflare Tunnel

Go to **Cloudflare Dashboard → Zero Trust → Networks → Tunnels → serverden-homelab → Configure → Public Hostnames → Add a public hostname**:

| Field | Value |
|-------|-------|
| Subdomain | `argocd` |
| Domain | `kovachev-projects.com` |
| Service Type | `HTTPS` |
| URL | `istio-gateway.istio-system.svc.cluster.local:443` |

Expand **Additional application settings → TLS**:

| Setting | Value |
|---------|-------|
| Origin Server Name | `argocd.kovachev-projects.com` |
| No TLS Verify | ✅ enabled |

**Why these settings are required:**

- **HTTPS (not HTTP):** The Istio Gateway port 80 has `httpsRedirect: true`. Sending HTTP causes an infinite redirect loop — Istio 301s to HTTPS, cloudflared follows, repeat.
- **Origin Server Name:** cloudflared connects to `istio-gateway.istio-system.svc.cluster.local` internally. Without this, it sends that as the TLS SNI, which doesn't match the cert (`*.kovachev-projects.com`). Setting the origin server name forces SNI to `argocd.kovachev-projects.com`, so Istio serves the right cert and routes via the correct VirtualService.
- **No TLS Verify:** cloudflared can't verify the cert chain for an internal cluster hostname. This is safe — the connection is entirely inside the cluster; public TLS is handled by Cloudflare.

Cloudflare auto-creates the CNAME DNS record when you save. Verify with:
```powershell
nslookup argocd.kovachev-projects.com 1.1.1.1
# Should resolve to Cloudflare anycast IPs
```

Open `https://argocd.kovachev-projects.com` — you should see the ArgoCD login page.

---

### Step 10 — Enable Istio mTLS STRICT mode

**Why:** By default Istio allows both plaintext and mTLS traffic between services (PERMISSIVE mode). Setting STRICT mode means every service-to-service call inside the mesh **must** use mTLS — plaintext is rejected. This closes the window where a pod bypassing the sidecar could talk to another service unencrypted.

Create `infrastructure/istio/peer-authentication.yaml`:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

> Applying this to the `istio-system` namespace sets it as the **mesh-wide default** — all namespaces inherit it.

Apply it:
```bash
kubectl apply -f infrastructure/istio/peer-authentication.yaml
```

Verify:
```bash
kubectl get peerauthentication -n istio-system
# NAME      MODE     AGE
# default   STRICT   ...
```

Verify ArgoCD still works at `https://argocd.kovachev-projects.com` after applying — if it does, mTLS is working correctly end-to-end.

---

### Step 11 — Commit Phase 1 to Git

All Phase 1 manifests are now ready to commit.

```bash
git add .
git commit -m "feat: Phase 1 foundation

- Disable Traefik, install open-iscsi
- Bootstrap ArgoCD v9.4.15 (server.insecure=true, Istio handles TLS)
- Install Sealed Secrets
- Install MetalLB with IP pool 10.10.10.240-10.10.10.250
- Install Istio (base, istiod, gateway) — service: istio-gateway, IP: 10.10.10.240
- Label namespaces for sidecar injection (infrastructure, monitoring, staging, production)
- Install cert-manager with Cloudflare DNS-01, wildcard cert *.kovachev-projects.com
- Deploy Cloudflare Tunnel (cloudflared v2026.3.0, 2 replicas)
- Install Longhorn v1.10.2 (2 replicas, default StorageClass)
- Expose ArgoCD via Istio Gateway + Cloudflare Tunnel
- Enable Istio mTLS STRICT mode cluster-wide

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
git push
```

---

---

## Phase 2 — GitOps Control Plane

**Goal:** ArgoCD watches this Git repo and owns the cluster state. No more manual `kubectl apply`.

We use the **App-of-Apps** pattern:
- A single root `Application` is applied manually once
- It watches `gitops/apps/` in the repo
- That directory contains one `Application` manifest per platform component
- ArgoCD creates and syncs all child apps automatically

```
gitops/
├── root-app.yaml          ← applied manually once (bootstraps everything)
└── apps/
    ├── cert-manager-config.yaml
    ├── istio-config.yaml
    ├── cloudflare.yaml
    └── argocd-config.yaml
```

### Step 12 — Scaffold the GitOps structure

All files are already in the repo. Apply the root app once to hand control to ArgoCD.

> Run from your **local Windows machine**.

```powershell
kubectl apply -f gitops/root-app.yaml
```

Watch ArgoCD create and sync all child apps:
```bash
kubectl get applications -n argocd
# All apps should go: OutOfSync → Synced
```

Open `https://argocd.kovachev-projects.com` — you should see the root app and all child apps in the UI, all `Healthy` and `Synced`.

> **Note:** The `argocd-config` app manages the ArgoCD VirtualService. If it shows a sync error about namespace mismatch, that is expected — the VirtualService lives in the `argocd` namespace and is already applied. ArgoCD will adopt it on the first sync.

### Step 13 — Verify GitOps is working

Make a test change to confirm the GitOps loop works end-to-end:

1. Edit any manifest in `infrastructure/` on your local machine
2. `git commit -m "test: gitops loop" && git push`
3. Watch ArgoCD detect the drift and auto-sync within ~3 min (default poll interval)
4. Revert the change and push again

If auto-sync triggers correctly, Phase 2 is complete.

---

## Phase 3 — Observability (LGTM + Alloy)

| Component | Role |
|-----------|------|
| **Mimir** | Metrics backend (replaces Prometheus) |
| **Loki** | Log aggregation |
| **Tempo** | Distributed tracing (replaces Jaeger) |
| **Grafana** | Unified UI — all three datasources |
| **Alloy** | Unified collector — scrapes metrics, tails logs, receives traces (replaces Promtail + Prometheus scraper) |
| **Kiali** | Istio mesh topology, points at Mimir + Tempo |

*Coming after Phase 2 is complete*

---

## Phase 4 — CI/CD

*Coming after Phase 3 is complete*

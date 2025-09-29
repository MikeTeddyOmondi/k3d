# K3D MacOS

⚡ This setup gives you a **Cilium-powered macOS k3d cluster** with Traefik ingress mapped to your laptop’s ports.

Let’s make a **k3d cluster config optimized for macOS**, with:

- **Cilium** as the CNI (no Flannel).
- **Traefik** as ingress (via Helm, not bundled).
- Proper **port mappings (80/443)** to the macOS host.

---

### 📄 `k3d-macos.yaml`

```yaml
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: macos-cluster
servers: 1
agents: 2
ports:
  - port: 80:80 # HTTP ingress
    nodeFilters:
      - loadbalancer
  - port: 443:443 # HTTPS ingress
    nodeFilters:
      - loadbalancer
kubeAPI:
  host: "127.0.0.1"
  hostPort: "6443"
options:
  k3s:
    extraArgs:
      - arg: "--disable=traefik"
        nodeFilters:
          - server:0
      - arg: "--flannel-backend=none"
        nodeFilters:
          - server:0
      - arg: "--disable-kube-proxy"
        nodeFilters:
          - server:0
```

Create the cluster:

```bash
k3d cluster create --config k3d-macos.yaml
```

---

### 🚀 Step 1: Install Cilium

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

helm upgrade --install cilium cilium/cilium \
  --namespace kube-system \
  --set k8sServiceHost=k3d-macos-cluster-server-0 \
  --set k8sServicePort=6443 \
  --set kubeProxyReplacement=true \
  --set hostServices.enabled=false \
  --set externalIPs.enabled=true \
  --set nodePort.enabled=true \
  --set loadBalancer.l7.backend=envoy \
  --set ipam.mode=kubernetes
```

Check:

```bash
kubectl -n kube-system get pods -l k8s-app=cilium
```

---

### 🚀 Step 2: Install Traefik

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
kubectl create namespace traefik

helm install traefik traefik/traefik \
  --namespace=traefik \
  --set service.type=LoadBalancer \
  --set ingressClass.enabled=true \
  --set ingressClass.isDefaultClass=true \
  --set "ports.web.http.redirections.entryPoint.to=websecure" \
  --set "ports.web.http.redirections.entryPoint.scheme=https" \
  --set "ports.web.http.redirections.entryPoint.permanent=true"

```

Now Traefik is exposed on `127.0.0.1:80` and `127.0.0.1:443`.

### Configure Cilium LoadBalancer IP Pool

After installing Traefik, configure Cilium to assign an external IP to the LoadBalancer service:

**`cilium-lb-ippool.yaml`**

```yaml
# Create a safer pool that explicitly handles only HTTP/HTTPS

apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: default-pool
spec:
  blocks:
    - cidr: YOUR_VPS_IP/32 # Replace with your VPS public IP
  serviceSelector:
    matchExpressions:
      - key: io.kubernetes.service.namespace
        operator: In
        values: ["traefik"]
```

> This limits the IP pool to only the `traefik` namespace, preventing SSH interference.

**Quick setup:**

```bash
# Get your VPS public IP
curl -s ifconfig.me

# Create and apply the IP pool (replace YOUR_VPS_IP with the actual IP)
kubectl apply -f - <<EOF
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: default-pool
spec:
  blocks:
    - cidr: $(curl -s ifconfig.me)/32
EOF
```

Get your VPS IP and apply:

```bash
# Get your VPS public IP
curl -s ifconfig.me

# Edit the file and replace YOUR_VPS_PUBLIC_IP with the actual IP
# Example: if IP is 203.0.113.55, use 203.0.113.55/32

kubectl apply -f cilium-lb-ippool.yaml
```

**Alternative: Use your local network IP if behind NAT:**

If your VPS uses a private IP internally:

```bash
# Get the node's IP
kubectl get nodes -o wide

# Use that IP in the pool, for example:
# - cidr: 192.168.1.100/32
```

**Verify the external IP assignment:**

```bash
# Watch the service until EXTERNAL-IP appears (should take 5-10 seconds)
kubectl get svc -n traefik traefik -w

# Expected output:
# NAME      TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
# traefik   LoadBalancer   10.43.98.191   203.0.113.55     80:30467/TCP,443:32067/TCP
```

Once the EXTERNAL-IP appears, Traefik is accessible on standard ports 80 and 443 at your VPS IP address.

---

### 🚀 Step 3: Test with a sample app

```bash
kubectl create deployment whoami --image=traefik/whoami --port=80
kubectl expose deployment whoami --port=80 --target-port=80
```

Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami
spec:
  rules:
    - host: whoami.localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f whoami-ingress.yaml
```

Add to `/etc/hosts`:

```
127.0.0.1 whoami.localhost
```

Test:

```
curl http://whoami.localhost
```

---

## Deploying a web application

Having already got **Traefik + Cilium** running on your `k3d` cluster, so we can deploy your `ranckosolutionsinc/sticky-notes-app:v2.0.0` container with a **Deployment**, expose it with a **Service**, and route traffic via an **Ingress**.

Here’s the full set of YAMLs you can apply:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sticky-notes
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sticky-notes
  template:
    metadata:
      labels:
        app: sticky-notes
    spec:
      containers:
        - name: sticky-notes
          image: ranckosolutionsinc/sticky-notes-app:v2.0.0
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: sticky-notes-service
  namespace: default
spec:
  selector:
    app: sticky-notes
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sticky-notes-ingress
  namespace: default
spec:
  ingressClassName: traefik # ✅ use this instead of the annotation
  rules:
    - host: sticky-notes.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sticky-notes-service
                port:
                  number: 80
```

---

### 🔧 Steps

1. Save the YAML as `sticky-notes.yaml`.

2. Apply it:

   ```bash
   kubectl apply -f sticky-notes.yaml
   ```

3. Add a host entry (since you’re using `sticky-notes.local`):

   ```bash
   sudo -- sh -c "echo '127.0.0.1 sticky-notes.local' >> /etc/hosts"
   ```

4. Access your app:

   ```
   http://sticky-notes.local
   ```

---

## Setting up TLS for the web application

Got it ✅ here’s the **short version** to get HTTPS with Traefik on `sticky-notes.local`:

---

### 1. Generate a local cert (with mkcert)

```bash
brew install mkcert nss
mkcert -install
mkcert sticky-notes.local
```

This gives you two files:

- `sticky-notes.local.pem` (cert)
- `sticky-notes.local-key.pem` (key)

---

### 2. Create a Kubernetes TLS Secret

```bash
kubectl create secret tls sticky-notes-tls \
  --cert=sticky-notes.local.pem \
  --key=sticky-notes.local-key.pem
```

---

### 3. Update your Ingress

Add TLS section:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sticky-notes-ingress
  namespace: default
spec:
  ingressClassName: traefik
  rules:
    - host: sticky-notes.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sticky-notes-service
                port:
                  number: 80
  tls:
    - hosts:
        - sticky-notes.local
      secretName: sticky-notes-tls # created from mkcert
```

---

### 4. Test

Open:

```
https://sticky-notes.local
```

Your browser should trust it (thanks to mkcert root CA).

---

## Using CertManager

Using **cert-manager** makes this way cleaner because you won’t have to generate or rotate certs manually:

---

### 1. Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml
```

(wait until pods in `cert-manager` namespace are `Running`)

---

### 2. Create a local CA Issuer (for development)

For local clusters (k3d, kind, etc.), you can use a **self-signed issuer**:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
```

Apply it:

```bash
kubectl apply -f cluster-issuer.yaml
```

---

### 3. Update your Ingress for TLS with cert-manager

Instead of manually creating a secret, cert-manager will handle it:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sticky-notes-ingress
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-cluster-issuer
spec:
  ingressClassName: traefik
  rules:
    - host: sticky-notes.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sticky-notes-service
                port:
                  number: 80
  tls:
    - hosts:
        - sticky-notes.local
      secretName: sticky-notes-tls
```

cert-manager will automatically create the `sticky-notes-tls` secret with a signed cert.

---

### 4. Access app

```
https://sticky-notes.local
```

You’ll see HTTPS, and cert-manager keeps it valid (even rotating when near expiry).

---

✨ Improvement over mkcert/manual TLS:

- No manual secret creation.
- Automatic renewal.
- Same config works in production with Let’s Encrypt (just swap issuer).

---

## Utilising Lets Encrypt (Prod)

Here’s the **Let’s Encrypt ClusterIssuer** version for production use with Traefik + cert-manager:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: info@mail.locci.cloud
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
```

Then in your **Ingress**:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```

cert-manager will:

- Use Let’s Encrypt ACME challenge via Traefik.
- Generate a valid cert (trusted in browsers).
- Auto-renew when expiry is near.

👉 With this, dev = self-signed issuer, prod = Let’s Encrypt issuer.
Same ingress config, just swap the `cluster-issuer`.

---

## Dashboard Using Kite

Pull the image

```bash
docker pull ghcr.io/zxh326/kite:latest
```

Start the container on the `host` network

```bash
docker run -d --name kite --network host \
  -v ~/.kube/config:/root/.kube/config:ro \
  ghcr.io/zxh326/kite:latest
```

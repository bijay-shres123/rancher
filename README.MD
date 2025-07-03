
# Rancher on k3s with NGINX Ingress & Let’s Encrypt SSL — Step-by-Step Guide

---

### Prerequisites

* AlmaLinux server (clean setup)
* Domain `rancher.yourdomain.com` pointing to your server IP (e.g., yourip : 12x.34.56.xx)
* Root or sudo access to server
* `kubectl`, `helm` installed

---

## Step 1: Install k3s (without Traefik)

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -
```

Verify k3s running:

```bash
sudo kubectl get nodes
```

---

## Step 2: Install NGINX Ingress Controller

```bash
kubectl create namespace ingress-nginx

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx
```

Wait until ingress-nginx pods are `Running`:

```bash
kubectl get pods -n ingress-nginx
```

---

## Step 3: Install cert-manager

```bash
kubectl create namespace cert-manager

helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.14.2 \
  --set installCRDs=true
```

Verify cert-manager pods are running:

```bash
kubectl get pods -n cert-manager
```

---

## Step 4: Create ClusterIssuer for Let’s Encrypt

Create a file `cluster-issuer.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your-email@domain.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

Apply it:

```bash
kubectl apply -f cluster-issuer.yaml
```

Check status:

```bash
kubectl describe clusterissuer letsencrypt-prod
```

Make sure status is `Ready=True`.

---

## Step 5: Install Rancher with Helm

Create namespace:

```bash
kubectl create namespace cattle-system
```

Add Rancher repo and update:

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

Install Rancher:

```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.yourdomain.pro \
  --set ingress.ingressClassName=nginx \
  --set replicas=1 \
  --set certmanager.install=false \
  --set ingress.extraAnnotations."cert-manager\.io/cluster-issuer"=letsencrypt-prod
```

---

## Step 6: Verify Rancher Ingress Annotations and TLS

Check ingress annotations:

```bash
kubectl get ingress rancher -n cattle-system -o yaml | grep -A 10 annotations
```

Ensure **only**:

```yaml
cert-manager.io/cluster-issuer: letsencrypt-prod
```

is set (remove `cert-manager.io/issuer` or `cert-manager.io/issuer-kind` if present).

---

## Step 7: Monitor Certificate Issuance

Check Certificate resource created by Rancher:

```bash
kubectl get certificate tls-rancher-ingress -n cattle-system
kubectl describe certificate tls-rancher-ingress -n cattle-system
```

Make sure status condition is `Ready=True`.

Check TLS secret:

```bash
kubectl get secret tls-rancher-ingress -n cattle-system -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer -subject -dates
```

Issuer should be Let's Encrypt, e.g.:

```
issuer= /C=US/O=Let's Encrypt/CN=R3
```

---

## Step 8: Access Rancher UI securely

Open in browser:

```
https://rancher.yourdomain.pro
```

You should see a valid SSL certificate issued by Let's Encrypt.

---

# Debugging Tips

---

### 1. **If certificate shows “Kubernetes Ingress Controller Fake Certificate” or similar:**

* Check the certificate secret contents (see Step 7)
* Ensure correct issuer is set on Certificate resource (ClusterIssuer letsencrypt-prod)
* Ensure ingress annotations only have `cert-manager.io/cluster-issuer: letsencrypt-prod`

---

### 2. **If Certificate resource not found or not ready:**

```bash
kubectl get certificate -n cattle-system
kubectl describe certificate -n cattle-system
kubectl logs -n cert-manager -l app=cert-manager --tail=50
```

Look for errors about issuer missing, challenges failing, or misconfigurations.

---

### 3. **If cert-manager logs show `issuer.cert-manager.io "rancher" not found`:**

* Check and remove conflicting ingress annotations:

```bash
kubectl annotate ingress rancher -n cattle-system cert-manager.io/issuer- cert-manager.io/issuer-kind- --overwrite
kubectl annotate ingress rancher -n cattle-system cert-manager.io/cluster-issuer=letsencrypt-prod --overwrite
```

Delete old certificate and secret:

```bash
kubectl delete certificate tls-rancher-ingress -n cattle-system --ignore-not-found
kubectl delete secret tls-rancher-ingress -n cattle-system --ignore-not-found
```

---

### 4. **If ACME HTTP-01 challenge fails:**

* Verify DNS A record points to your server IP
* Verify ports 80 and 443 are open and reachable from internet
* Verify nginx ingress is properly installed and serving HTTP on port 80

---

# Bonus: Useful commands

```bash
kubectl get pods -n cert-manager
kubectl get pods -n ingress-nginx
kubectl get ingress -n cattle-system
kubectl describe ingress rancher -n cattle-system
kubectl get secrets -n cattle-system
kubectl get certificates -n cattle-system
kubectl get clusterissuer letsencrypt-prod -o yaml
kubectl logs -n cert-manager -l app=cert-manager --tail=100
```


---

Glad you got the cert working! Feel free to ask if you want help automating or optimizing any step.

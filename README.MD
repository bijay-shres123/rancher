
# Rancher on k3s with NGINX Ingress & Let’s Encrypt SSL — Step-by-Step Guide

---

### Prerequisites

* AlmaLinux server (clean setup)
* Domain `rancher.yourdomain.com` pointing to your server IP (e.g., yourip : 12x.34.56.xx)
use dig command to verify:
```bash
dig +short rancher.<yourdomain>.pro
```
![dig status](images/verify.png)
* Root or sudo access to server


---
## Step 0: Install & Enable Firewalld (Required for SSL and Ingress)
```bash

sudo dnf update -y

sudo dnf install -y firewalld
sudo systemctl enable firewalld
sudo systemctl start firewalld
```
verify:
```bash
sudo systemctl status firewalld
```

![firewalld status](images/firewalld_status.png)
## Configure Firewalld (Before Cert Manager or Ingress)
### Open necessary ports
```bash
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-masquerade

# Add flannel interface to trusted zone
sudo firewall-cmd --permanent --zone=trusted --add-interface=cni0
sudo firewall-cmd --permanent --zone=trusted --add-interface=flannel.1

# If you're using a different CNI, check with:
ip link show | grep -E "(cni|flannel)"
```

### Apply changes and restart
```bash
sudo firewall-cmd --reload
sudo systemctl restart firewalld  # 🔥 IMPORTANT!
```

check list of open ports 
```bash
sudo firewall-cmd --list-ports
```
verify:

![open port list](images/list_open_ports.png)


## Step 1: Install k3s (without Traefik)

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik --write-kubeconfig-mode=644 --kube-apiserver-arg=audit-log-path=/var/log/audit.log" sh -
```

Verify k3s running:
```bash
systemctl status k3s
```
verify:
![k3sstatus](images/k3s_status_check.png)

```bash
# Ensure the directory exists (idempotent)
mkdir -p ~/.kube

# Copy the config file with sudo
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

# Set correct permissions (note the ~)
chmod 600 ~/.kube/config

# Set the KUBECONFIG environment variable (note the ~)
export KUBECONFIG=~/.kube/config

# Make KUBECONFIG persistent for future sessions (optional but recommended)
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc # Apply changes immediately
```

```bash
kubectl get nodes
```
verify:
![kubectl status](images/kubectl_get_nodes.png)

Connect to lens( 👓 optional)
```bash
cat ~/.kube/config 
#on the config file replace 
server: https://127.0.0.1:6443 -> https://<your-server-ip>:6443 
```

---



## Step 2: Install NGINX Ingress Controller

```bash
#create a namespace for ingress-nginx

kubectl create namespace ingress-nginx
# install helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx
```

Wait until ingress-nginx pods are `Running`:

```bash
kubectl get pods -n ingress-nginx
```
verify:
![nginx ingress pods](images/ingress_nginx_get_pods.png)
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
verify:
![cert manager pods](images/pods_cert_manager.png)
---

## Step 4: Create ClusterIssuer for Let’s Encrypt

Create a file `cluster-issuer.yaml`:
#🔥 IMPORTANT use your valid email!
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your-email@gmail.com 
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
verify:
![ACME status](images/ACME.png)
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
# 🫲Note: Replace rancher.yourdomain.pro with your actual domain
```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.<yourdomain>.pro \
  --set ingress.ingressClassName=nginx \
  --set replicas=1 \
  --set certmanager.install=false \
  --set ingress.extraAnnotations."cert-manager\.io/cluster-issuer"=letsencrypt-prod \
  --set-string ingress.extraAnnotations."cert-manager\.io/issuer"=null \
  --set-string ingress.extraAnnotations."cert-manager\.io/issuer-kind"=null
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

🚨CRITICAL STEP BEFORE MOVING FORWARD


(remove `cert-manager.io/issuer` or `cert-manager.io/issuer-kind`).

```yaml
kubectl edit ingress rancher -n cattle-system

#remove these two annotations
cert-manager.io/issuer
cert-manager.io/issuer-kind
```
---

## Step 7: Monitor Certificate Issuance

Check Certificate resource created by Rancher:

```bash
kubectl get certificate tls-rancher-ingress -n cattle-system
kubectl describe certificate tls-rancher-ingress -n cattle-system
```
verify:
![ingress](images/get_certificate.png)
![ingress](images/cert_detail.png)
Make sure status condition is `Ready=True`.

Check TLS secret:

```bash
kubectl get secret tls-rancher-ingress -n cattle-system -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer -subject -dates
```

Issuer should be Let's Encrypt, e.g.:

```
issuer= /C=US/O=Let's Encrypt/CN=R3 # ✅ Must verify
```
verify:
![ingress](images/rancher_ingress_check.png)
---

## Step 8: Access Rancher UI securely

Open in browser:

```
https://rancher.<yourdomain>.pro
```

You should see a valid SSL certificate issued by Let's Encrypt.

---
## Generate the default password for Rancher Dashboard
```bash
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'
```
### if default rancher password is not available
```bash
kubectl exec -n cattle-system $(kubectl get pods -n cattle-system -l app=rancher -o jsonpath='{.items[0].metadata.name}') -- reset-password
```

### Debugging Tips

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

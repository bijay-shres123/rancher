nftable as the default firewall active / 
# Update system and install basic tools
dnf update -y
dnf install -y curl vim wget git firewalld
systemctl enable firewalld --now


# Install k3s with Traefik disabled
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -


# Verify k3s installation
sudo systemctl status k3s
k3s --version

# Enable kubectl access
mkdir -p ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG=~/.kube/config

# Confirm it's working
kubectl get nodes


# Connect to lens
cat ~/.kube/config
replace server: https://127.0.0.1:6443 -> https://<your-ip>:6443 in the config file

# nginx ingress controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml

# Install Cert-Manager for SSL
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
kubectl create namespace cert-manager

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.14.2

kubectl apply --validate=false -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.2/cert-manager.crds.yaml


helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.codekarma.pro \
  --set ingress.ingressClassName=nginx \
  --set replicas=1



# Wait for all pods
kubectl get pods --namespace cert-manager --watch

# Create ClusterIssuer (Let's Encrypt)

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your-email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx

          
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update


kubectl create namespace cattle-system








Also, check your Rancher ingress TLS section:
bash
Copy
Edit
kubectl get ingress -n cattle-system rancher -o yaml
You should see:

yaml
Copy
Edit
tls:
- hosts:
  - rancher.codekarma.pro
  secretName: tls-rancher-ingress
and annotations like:

yaml
Copy
Edit
cert-manager.io/cluster-issuer: letsencrypt-prod


helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.codekarma.pro \
  --set ingress.ingressClassName=nginx \
  --set replicas=1 \
  --set certmanager.install=false \
  --set ingress.tls.source=cert-manager \
  --set ingress.extraAnnotations."cert-manager\.io/cluster-issuer"=letsencrypt-prod

---------------------



# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install cert-manager
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --set installCRDs=true
  
  Create ClusterIssuer Manually
If it doesn't exist, create one:

kubectl get clusterissuer
If you see nothing, then Rancher can’t request certificates via cert-manager.

# letsencrypt-prod.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: bijay.shres123@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
          ingressTemplate:
            metadata:
              annotations:
                nginx.ingress.kubernetes.io/rewrite-target: /

Check if Certificate Resource exists

kubectl get certificate -n cattle-system
You should see a Certificate resource — possibly named something like tls-rancher-ingress.

If not, cert-manager hasn't been triggered to create it.

✅ 2. Manually create a Certificate resource
If the Certificate isn't there or is misconfigured, create it manually like this:


# rancher-cert.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tls-rancher-ingress
  namespace: cattle-system
spec:
  secretName: tls-rancher-ingress
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: rancher.codekarma.pro
  dnsNames:
    - rancher.codekarma.pro

Apply it:

kubectl apply -f rancher-cert.yaml


# Install nginx ingress controller
kubectl create namespace ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx

# Install Rancher
kubectl create namespace cattle-system
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update

helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.codekarma.pro \
  --set ingress.ingressClassName=nginx \
  --set replicas=1 \
  --set certmanager.enabled=true \
  --set ingress.tls.source=secret \
  --set ingress.annotations."cert-manager\.io/cluster-issuer"=letsencrypt-prod


  check cert manager pod logs if you face any error with tls 

  cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rancher
  namespace: cattle-system
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "1800"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "1800"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - rancher.codekarma.pro
    secretName: tls-rancher-ingress
  rules:
  - host: rancher.codekarma.pro
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: rancher
            port:
              number: 80
EOF

# Verify ingress is created
kubectl get ingress -n cattle-system

# Now cert-manager should automatically create the certificate
# Wait a minute then check
sleep 60

# Check if certificate was auto-created
kubectl get certificate -n cattle-system


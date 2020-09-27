# Install and configure Kubernetes
## Install K3S
```bash
curl -sfL https://get.k3s.io | sh -
```

## Configure kubectl
```bash
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/
sudo chown tons.tons ~/.kube/k3s.yaml
export KUBECONFIG=$HOME/.kube/k3s.yaml
```

## Show nodes
```bash
kubectl get nodes
```

# Helm - The Kubernetes Package Manager
See https://helm.sh/docs/intro/quickstart/

# SSL - Lets Encrypt
* https://pascalw.me/blog/2019/07/02/k3s-https-letsencrypt.html
* https://www.thebookofjoel.com/k3s-cert-manager-letsencrypt

```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.1/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
kubectl create namespace cert-manager
helm install cert-manager jetstack/cert-manager --version 1.0.1 --namespace cert-manager

echo "apiVersion: cert-manager.io/v1beta1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: sebastianthegreatful@gmail.com
    privateKeySecretRef:
      name: prod-issuer-account-key
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - http01:
          ingress:
            class: traefik
" | kubectl apply --validate=false -f -

echo "apiVersion: cert-manager.io/v1beta1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: sebastianthegreatful@gmail.com
    privateKeySecretRef:
      name: staging-issuer-account-key
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    solvers:
      - http01:
          ingress:
            class: traefik
" | kubectl apply --validate=false -f -
```

# Dashboard
* https://rancher.com/docs/k3s/latest/en/installation/kube-dashboard/

## Install the Dashboard
```bash
GITHUB_URL=https://github.com/kubernetes/dashboard/releases
VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml
```

## Create Service Account
```bash
echo "apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
" | kubectl create -f -
```

## Create Cluster Role Binding
```bash
echo "apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
" | kubectl create -f -
```

## Get Access Token
```bash
kubectl -n kubernetes-dashboard describe secret admin-user-token | grep ^token
```

## Forward port to localhost
```bash
kubectl proxy
```

## Browse Dashboard
Open the below url in your favorite browser

`http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/logi`

# Helmfile - docker-compose for Kubernetes
https://github.com/roboll/helmfile#installation

# Chartmuseum - Helm Chart Repository
Export `HARTMUSEUM_AUTH_USER` and `HARTMUSEUM_AUTH_PASS` with your desired username and password

Use your favorite editor to save the below snippet in a file called helmfile.yaml
```yaml
releases:
  - name: chartmuseum
    namespace: chartmuseum
    chart: stable/chartmuseum
    version: 2.13.3
    values:
      - ingress:
          enabled: true
          annotations:
            kubernetes.io/ingress.class: traefik
            cert-manager.io/cluster-issuer: letsencrypt-prod
          hosts:
            - name: helm-charts.your-domain.com
              path: /
              tls: true
              tlsSecret: chartmuseum-tls
      - env:
          open:
            DISABLE_API: false
            STORAGE: local
          secret:
            BASIC_AUTH_USER: {{ requiredEnv "CHARTMUSEUM_AUTH_USER" }}
            BASIC_AUTH_PASS: {{ requiredEnv "CHARTMUSEUM_AUTH_PASS" }}
      - persistence:
          enabled: true
          accessMode: ReadWriteOnce
          size: 100M
```

## Install Chartmuseum
```bash
helmfile sync
```

## Install Chartmuseum helm plugin
```bash
helm plugin install https://github.com/chartmuseum/helm-push.git
```

# Install WhoAmI - A simple application which will return the hostname of the pods it's running on
This app can serve as a simple example of the relation between a `deployment`, `service` and `ingress` resource. For a more detailed explanation please see the below url.

`https://medium.com/@dwdraju/how-deployment-service-ingress-are-related-in-their-manifest-a2e553cf0ffb`

Use git to clone or fork the below reository

`https://github.com/tonsV2/whoami-mn`

Replace the `kubeContext`with the name of your context. Possible `default`

https://github.com/tonsV2/whoami-mn/blob/master/helmfile.yaml#L4

Replace the repository with the url of your Chartmuseum

`https://github.com/tonsV2/whoami-mn/blob/master/helmfile.yaml#L36`

## Push the chart to the repository
```bash
helm push src/main/helm tons --username $CHARTMUSEUM_AUTH_USER --password $CHARTMUSEUM_AUTH_PASS
```

## Deploy using helmfile
See https://github.com/tonsV2/whoami-mn#deploy
```bash
helmfile -e dev sync
```

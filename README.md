1. Deploy bookinfo application
2. Deploy istio ( gateway, virtualservice, destinationrules)

Test
1. Apply observability, track it using pro and grafana
2. kiali
$ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.30/samples/addons/kiali.yaml



kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.30/samples/addons/prometheus.yaml

kubectl rollout restart deployment/kiali -n istio-system


3. mTLS
4. Upgrading istio to a new version.

minikube start --driver=docker --nodes=2 --cpus=2 --memory=3072

## Argo CD with Istio mesh

This repo now includes declarative files to run Argo CD inside the current Istio mesh:

- argocd-install.yaml
- argocd-istio.yaml
- argocd-application.yaml

### 1) Install Argo CD in mesh-enabled namespace

```bash
kubectl apply -f argocd-install.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl rollout restart deployment argocd-server -n argocd
```

### 2) Expose Argo CD through Istio ingress gateway

```bash
kubectl apply -f argocd-istio.yaml
```

Set host resolution for local testing:

```bash
export INGRESS_IP=$(kubectl -n istio-system get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "$INGRESS_IP argocd.local" | sudo tee -a /etc/hosts
```

If you are using Minikube, you can use:

```bash
export INGRESS_IP=$(minikube ip)
```

Then open:

```bash
http://argocd.local
```

### 3) Register this repo as an Argo CD Application

Update the repo URL in argocd-application.yaml first:

```yaml
spec:
	source:
		repoURL: https://github.com/your-org/your-repo.git
```

Then apply:

```bash
kubectl apply -f argocd-application.yaml
```

### 4) Get initial Argo CD admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
echo
```

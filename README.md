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

or 

minikube service istio-ingressgateway -n istio-system --url

add argocd at the end of the url

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

## KEDA + Prometheus + Istio Autoscaling

This repo can use KEDA with Prometheus and Istio metrics to autoscale workloads based on:

- CPU utilization
- HTTP request rate observed by Istio

The current KEDA examples are in [Bookinfo-app/scaling.yaml](/Users/sajilpb/Documents/project/Istio/Bookinfo-app/scaling.yaml).

### How it works

Istio exports request metrics such as `istio_requests_total`.
Prometheus stores those metrics.
KEDA reads Prometheus queries and creates an HPA for the target workload.

For the `reviews` workloads, each `ScaledObject` uses two triggers:

- `cpu` trigger with `Utilization: 70`
- `prometheus` trigger using Istio request metrics

Example query used for `reviews-v2`:

```promql
sum(
	rate(istio_requests_total{
		reporter="destination",
		destination_workload_namespace="default",
		destination_workload="reviews-v2"
	}[1m])
)
```

This query means:

- use Istio destination-side request metrics
- look only at traffic received by `reviews-v2` in the `default` namespace
- convert the counter into requests per second over the last 1 minute
- sum all matching series into one total RPS value

### Important note about per-pod requests

The current query tracks total requests per workload, not true requests per pod.

To calculate requests per pod, Prometheus also needs replica metrics such as `kube_deployment_status_replicas`, usually provided by `kube-state-metrics`.

Without `kube-state-metrics`, the practical setup is:

- CPU scaling from Kubernetes resource metrics
- total workload request rate from Istio + Prometheus

### Prerequisites

Install KEDA:

```bash
kubectl apply -f https://github.com/kedacore/keda/releases/download/v2.15.1/keda-2.15.1.yaml
kubectl get pods -n keda
```

Install Prometheus from Istio addons:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.30/samples/addons/prometheus.yaml
kubectl get pods -n istio-system
```

Optional: install `kube-state-metrics` if you want true per-pod RPS calculations:

```bash
kubectl apply -f https://github.com/kubernetes/kube-state-metrics/releases/latest/download/standard.yaml
```

### Apply scaling objects

```bash
kubectl apply -f Bookinfo-app/scaling.yaml
kubectl get scaledobject -n default
kubectl get hpa -n default
```

KEDA creates an HPA for each `ScaledObject`.
Do not keep a separate manually managed HPA for the same workload, or both controllers will fight over `spec.replicas`.

### Test the metrics

You do not need to expose `reviews-v2` externally.
You can test it directly with a port-forward:

```bash
kubectl port-forward deploy/reviews-v2 9082:9080
```

Then generate traffic:

```bash
for i in {1..100}; do
	curl -s http://127.0.0.1:9082/reviews/0 > /dev/null
done
```

Then query Prometheus:

```bash
kubectl -n istio-system port-forward svc/prometheus 9090:9090
```

```bash
curl -G 'http://127.0.0.1:9090/api/v1/query' \
	--data-urlencode 'query=sum(rate(istio_requests_total{reporter="destination",destination_workload_namespace="default",destination_workload="reviews-v2"}[1m]))'
```

### Observe scaling

```bash
kubectl get scaledobject -n default
kubectl get hpa -n default -w
kubectl describe hpa keda-hpa-reviews-v2-scaledobject -n default
kubectl top pods -n default
```

### Summary

- Istio provides request metrics
- Prometheus stores and exposes those metrics
- KEDA uses CPU and Prometheus triggers to scale workloads
- current query is total workload RPS
- true per-pod RPS requires additional replica metrics from `kube-state-metrics`

# Setup Istio with Konvoy

## Installation

This installation guide was tested with the following components:

- D2iQ Konvoy v1.1.5
- Istio 1.3.0

Download and unpack Istio to your workstation:

```bash
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.3.0 sh -
cd istio-1.3.0
```

Create the namespace `istio-system`:

```bash
kubectl create namespace istio-system
```

Deploy the custom resources via `helm template` and `kubectl apply`:

```bash
helm template install/kubernetes/helm/istio-init \
  --name istio-init \
  --namespace istio-system | kubectl apply -f -
```

Also configure the Kiali add-on for Istio to use the web-based graphical user interface to view service graphs of the mesh and your Istio configuration objects. Additionally, you can use the Kiali Public API to generate graph data in the form of consumable JSON.

Documentation: [Istio - Kilali][istio-kiali]

- Define username and password for Kiali:

```bash
KIALI_USERNAME=`echo -n "admin" | base64`
KIALI_PASSPHRASE=`echo -n "admin" | base64`
NAMESPACE=istio-system
```

Now, create a secret for these credentials:

```yaml
cat <<EOF | tee /tmp/kiali-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: kiali
  namespace: $NAMESPACE
  labels:
    app: kiali
type: Opaque
data:
  username: $KIALI_USERNAME
  passphrase: $KIALI_PASSPHRASE
EOF

kubectl apply -f /tmp/kiali-secrets.yaml
```

Set Istio configuration parameters via `helm template` and apply the Istio app defintions via:

```bash
helm template install/kubernetes/helm/istio \
  --name istio \
  --namespace istio-system \
  --set kiali.enabled=true \
  --set "kiali.dashboard.jaegerURL=http://jaeger-query:16686" \
  --set "kiali.dashboard.grafanaURL=http://grafana:3000" \
  --values install/kubernetes/helm/istio/values-istio-demo.yaml | kubectl apply -f -
```

Check that all pods and services in the newly created `istio-system` are in the state `running` or `completed`:

```bash
kubectl get po -n istio-system

NAME                                      READY   STATUS      RESTARTS   AGE
grafana-59d57c5c56-bvqx6                  1/1     Running     0          60s
istio-citadel-5fc55bf54f-kdhqg            1/1     Running     0          57s
istio-cleanup-secrets-1.3.0-qslcj         0/1     Completed   0          78s
istio-egressgateway-774bcbb8bf-jrgw5      0/1     Running     0          61s
istio-galley-64f56bc4ff-h5c4d             1/1     Running     0          62s
istio-grafana-post-install-1.3.0-7tlwm    0/1     Completed   0          82s
istio-ingressgateway-5d6879d57d-pbrwn     0/1     Running     0          61s
istio-init-crd-10-1.3.0-kzp27             0/1     Completed   0          2m27s
istio-init-crd-11-1.3.0-mqcwq             0/1     Completed   0          2m26s
istio-init-crd-12-1.3.0-x9vzc             0/1     Completed   0          2m26s
istio-pilot-57df545d5c-s4pf2              1/2     Running     0          58s
istio-policy-5b48d95fbb-4snrm             2/2     Running     2          59s
istio-security-post-install-1.3.0-vwnpd   0/1     Completed   0          76s
istio-sidecar-injector-5f9655997b-pbzxm   1/1     Running     0          57s
istio-telemetry-665f555c68-wkcsz          2/2     Running     2          59s
istio-tracing-6bbdc67d6c-jdkrq            1/1     Running     0          56s
kiali-8c9d6fbf6-2bfk6                     1/1     Running     0          60s
prometheus-7d7b9f7844-qg65s               1/1     Running     0          58s
```

The Istio deployment also created a couple of services you can list them via:

```bash
kubectl get svc -n istio-system
```

## Port-Forwarding of Services

Without the need to expose each service, you can just do a port-forward for a quick test. In the following example, you access the port of Kiali directly. So you are able to view the web interface in the browser at [http://localhost:20001](http://localhost:20001).

```bash
kubectl -n istio-system port-forward \
  $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') \
  20001:20001
```

You can do the amse for other services like Grafana, Prometheus or Jager:
To port-forward Grafana:

```bash
# Grafana
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana \
  -o jsonpath='{.items[0].metadata.name}') 3000:3000

# Prometheus
kubectl -n istio-system port-forward \
  $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') \
  9090:9090

# Service Graph
kubectl -n istio-system port-forward \
  $(kubectl -n istio-system get pod -l app=servicegraph -o jsonpath='{.items[0].metadata.name}') \
  8088:8088

# Tracing/Jaeger
kubectl -n istio-system port-forward \
  $(kubectl -n istio-system get pod -l app=jaeger -o jsonpath='{.items[0].metadata.name}') \
  16686:16686
```

## Access Istio Ingress Gateway

Istio will also deploy an ingress gateway, you can list it via the following command:

```bash
kubectl get svc istio-ingressgateway -n istio-system

NAME                   TYPE           CLUSTER-IP    EXTERNAL-IP
istio-ingressgateway   LoadBalancer   10.0.35.108   a2c4014287c3544939122bf1a3d6a617-1341664630.us-west-2.elb.amazonaws.com
```

**Note:** The ingress gateway will not serve any response on port `80` until a Istio gateway was deployed. This will be done as part of the demo in the following section:

- [Demo: BookInfo](../bookinfo/)

[istio-kiali]: https://istio.io/docs/tasks/telemetry/kiali/

# Security

**Note:** The following guide assumes that the BookInfo demo was deployed successfully. See: [Demo: BookInfo](../bookinfo/)

## Secure Ingress Gateway

Documentation: [Istio / Securing Gateways with HTTPS](https://istio.io/docs/tasks/traffic-management/secure-ingress/#configure-traffic-for-the-bookinfo-com-host)

Generate a self-signed certificate for an example VHost named `bookinfo.com`:

```bash
git clone https://github.com/nicholasjackson/mtls-go-example

pushd mtls-go-example
./generate.sh bookinfo.com test123
mkdir ~+1/bookinfo.com && mv 1_root 2_intermediate 3_application 4_client ~+1/bookinfo.com
popd
```

Put the newly created certs into a Kubernetes secret:

```bash
kubectl create -n istio-system secret tls istio-ingressgateway-bookinfo-certs --key mtls-go-example/bookinfo.com/3_application/private/bookinfo.com.key.pem --cert mtls-go-example/bookinfo.com/3_application/certs/bookinfo.com.cert.pem
```

Redeploy the Istio ingress gateway with the referecend secrets. You can do this via the `helm template` command:

```bash
helm template install/kubernetes/helm/istio/ --name istio-ingressgateway --namespace istio-system -x charts/gateways/templates/deployment.yaml --set 'gateways.istio-egressgateway.enabled=false' \
--set 'gateways.istio-ingressgateway.secretVolumes[0].name=ingressgateway-certs' \
--set 'gateways.istio-ingressgateway.secretVolumes[0].secretName=istio-ingressgateway-certs' \
--set 'gateways.istio-ingressgateway.secretVolumes[0].mountPath=/etc/istio/ingressgateway-certs' \
--set 'gateways.istio-ingressgateway.secretVolumes[1].name=ingressgateway-ca-certs' \
--set 'gateways.istio-ingressgateway.secretVolumes[1].secretName=istio-ingressgateway-ca-certs' \
--set 'gateways.istio-ingressgateway.secretVolumes[1].mountPath=/etc/istio/ingressgateway-ca-certs' \
--set 'gateways.istio-ingressgateway.secretVolumes[2].name=ingressgateway-bookinfo-certs' \
--set 'gateways.istio-ingressgateway.secretVolumes[2].secretName=istio-ingressgateway-bookinfo-certs' \
--set 'gateways.istio-ingressgateway.secretVolumes[2].mountPath=/etc/istio/ingressgateway-bookinfo-certs' > \
/tmp/istio-ingressgateway.yaml
```

Deploy the new ingress gateway via `kubectl apply` and verfiy that the certs are available at the desired mount point:

```bash
# deploy
kubectl apply -f /tmp/istio-ingressgateway.yaml

# verify
kubectl exec -it -n istio-system $(kubectl -n istio-system get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}') -- ls -al /etc/istio/ingressgateway-bookinfo-certs

total 0
drwxrwxrwt. 3 root root 120 Mar  4 15:54 .
drwxr-xr-x. 1 root root 115 Mar  4 15:54 ..
drwxr-xr-x. 2 root root  80 Mar  4 15:54 ..2019_03_04_15_54_40.722502407
lrwxrwxrwx. 1 root root  31 Mar  4 15:54 ..data -> ..2019_03_04_15_54_40.722502407
lrwxrwxrwx. 1 root root  14 Mar  4 15:54 tls.crt -> ..data/tls.crt
lrwxrwxrwx. 1 root root  14 Mar  4 15:54 tls.key -> ..data/tls.key
```

Now deploy a new Istio gatway for the BookInfo app. This gateway will listen on port `443` and will reference the self-signed certificates.

```yaml
cat <<EOF | tee /tmp/secure-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-secure-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https-bookinfo
      protocol: HTTPS
    tls:
      mode: SIMPLE
      serverCertificate: /etc/istio/ingressgateway-bookinfo-certs/tls.crt
      privateKey: /etc/istio/ingressgateway-bookinfo-certs/tls.key
    hosts:
    - "*"
EOF

kubectl apply -f /tmp/secure-gateway.yaml
kubectl get gateway
```

You can verify that the traffic is routed correctly, by accessing your Public Agent IP with `HTTPS`:

```bash
curl -k --head https://$PUBLIC_NODE_IP/productpage
HTTP/2 200
content-type: text/html; charset=utf-8
content-length: 5723
server: envoy
date: Mon, 04 Mar 2019 15:56:10 GMT
x-envoy-upstream-service-time: 18
```

## End-User Auth on Ingress Gateway

The following example shows how to use end-user authentication with Istio based on JSON Web Tokens (JWT).

Documentation: [Istio / Securing Gateways with HTTPS](https://istio.io/docs/tasks/traffic-management/secure-ingress/#configure-end-user-authentication-on-ingress-gateway)

Verify that you can still access the BookInfo demo application via `HTTP`:

```bash
curl --silent --head http://$PUBLIC_NODE_IP/productpage
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 5719
server: envoy
date: Mon, 04 Mar 2019 16:39:27 GMT
x-envoy-upstream-service-time: 26
```

Setup an Istio policy called `ingressgateway` that references a JSON Web Key Set (JWKS) found in the Istio documentation:

```yaml
cat <<EOF | tee /tmp/ingress-policy.yml
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "ingressgateway"
  namespace: istio-system
spec:
  targets:
  - name: istio-ingressgateway
  origins:
  - jwt:
      issuer: "testing@secure.istio.io"
      jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.0/security/tools/jwt/samples/jwks.json"
  principalBinding: USE_ORIGIN
EOF

kubectl apply -f /tmp/ingress-policy.yml
```

When the new Istio policy was applied, you should see that the previous `curl` command should now fail with a `401 Unauthorized` response:

```bash
# fails
curl --silent --head http://$PUBLIC_NODE_IP/productpage
HTTP/1.1 401 Unauthorized
content-length: 29
content-type: text/plain
date: Mon, 04 Mar 2019 16:47:51 GMT
server: envoy
```

Now let's specify an example token in the HTTP header via `curl`. The curl command shoudl now succeed:

```bash
# set token
TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.0/security/tools/jwt/samples/demo.jwt -s)

# succeeds
curl --header "Authorization: Bearer $TOKEN" --silent --head http://$PUBLIC_NODE_IP/productpage
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 4415
server: envoy
date: Mon, 04 Mar 2019 16:56:28 GMT
x-envoy-upstream-service-time: 17
```

You can reset the ingress policy via:

```bash
# clean-up
kubectl delete -f /tmp/ingress-policy.yml
```

## Enable mTLS

This example enables Mutual TLS between all microservices of the BookInfo demo app.

Documentation: [Istio / Authentication Policy](https://istio.io/docs/tasks/security/authn-policy/)

With the following configuration, you will disable `mtls` explicitly for the ingress gateway, but you will enable it for every other peer of the service mesh:

```yaml
# globally enabling mutual TLS
cat <<EOF | tee /tmp/enable-mtls.yml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: istio-ingressgateway-disable-mtls
spec:
  host: istio-ingressgateway.svc.cluster.local
  trafficPolicy:
    tls:
      mode: DISABLE
---
apiVersion: "authentication.istio.io/v1alpha1"
kind: "MeshPolicy"
metadata:
  name: "default"
spec:
  peers:
  - mtls: {}
EOF

# deploy
kubectl apply -f /tmp/enable-mtls.yml
```

Since `mtls` is now enabled/required, we need to route all microservices through it, by defining the following destination rule:

```bash
# set rules for mtls
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```

You can reset the `mtls` policy via:

```bash
# clean-up
kubectl delete -f /tmp/enable-mtls.yml
```
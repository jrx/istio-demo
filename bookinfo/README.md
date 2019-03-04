# Demo: BookInfo

You can deploy the demo application that comes with Istio to see if your setup is working as expected.

Documentation: [Istio / Bookinfo Application](istio-bookinfo)

Switch to the default namespace:

```bash
kubectl config set-context $(kubectl config current-context) --namespace=default
```

You can use the option for automatic-sidecar-injection if you don’t want to run anything else in the default namespace. If you want to keep the apps separate that should belong to an Istio service mesh, the recommendation is to create a separate namespace and enable auto-sidecar-injection only for that namespace, or not use auto-sidecar-injection at all. 

Enable automatic sidecar injection for the default namespace with:

```bash
kubectl label namespace default istio-injection=enabled
```

Deploy the demo app:

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

Verify that the pods are Running and have a sidecar attached (`2/2`):

```bash
kubectl get po,svc
NAME                                  READY   STATUS    RESTARTS   AGE
pod/details-v1-68c7c8666d-cpsg6       2/2     Running   0          5h18m
pod/productpage-v1-54d799c966-tpdsx   2/2     Running   0          5h18m
pod/ratings-v1-8558d4458d-lmq24       2/2     Running   0          5h18m
pod/reviews-v1-cb8655c75-lsq5f        2/2     Running   0          5h18m
pod/reviews-v2-7fc9bb6dcf-pxht4       2/2     Running   0          5h18m
pod/reviews-v3-c995979bc-d5p8m        2/2     Running   0          5h18m
```

Deploy the gateway to enable the traffic flow from your Edge-LB pool configuration:

```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get gateway
```

Confirm that the BookInfo app is now reachable on your Public Node:

```bash
curl --head http://$PUBLIC_NODE_IP/productpage
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 5719
server: envoy
date: Fri, 08 Feb 2019 15:42:50 GMT
x-envoy-upstream-service-time: 34
```

Deploy the destination rules:

```bash
kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
kubectl get destinationrules -o yaml
```

With this demo working, you can follow the other examples provided in [Istio Documentation](istio-bookinfo) or the guides below:

* [Demo: Intelligent Routing](../routing/)
* [Demo: Security](../security/)

## Clean-Up

If you want to remove the BookInfo demo from your cluster, you can run the following commands:

```bash
# clean-up
samples/bookinfo/platform/kube/cleanup.sh

# confirm
kubectl get virtualservices
kubectl get destinationrules
kubectl get gateway
kubectl get pods
```

[istio-bookinfo]: https://istio.io/docs/examples/bookinfo/#if-you-are-running-on-kubernetes
# Intelligent Routing

**Note:** The following guide assumes that the BookInfo demo was deployed successfully. See: [Demo: BookInfo](../bookinfo/)

## Request Routing

Documentation: [Istio / Configuring Request Routing](https://istio.io/docs/tasks/traffic-management/request-routing/)

```bash
# route reviews to v1
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml

kubectl get destinationrules -o yaml

# route reviews to v2 for user jason
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml

# clean-up
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

## Fault Injection

Documentation: [Istio / Fault Injection](https://istio.io/docs/tasks/traffic-management/fault-injection/)

```bash
# apply version routing
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml

# delay fault for user jason
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml

# http abort fault for user jason
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml

# clean-up
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

## Traffic Shifting

Documentation: [Istio / Traffic Shifting](https://istio.io/docs/tasks/traffic-management/traffic-shifting/)

```bash
# apply version routing
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml

# transfer 50% from v1 to v3
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml

# route 100 % to v3
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml

# clean-up
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

## Next

To see the security features of Istio in action, try the following guide:

* [Demo: Security](../security/)
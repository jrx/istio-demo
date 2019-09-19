# Istio

Istio is the control plane implementation of a service mesh. A service mesh offers consistent discovery, security, tracing, monitoring and failure handling for microservices without the need for a shared asset such as an API gateway.  Some of the capabilities of Istio include automatic load balancing for HTTP, gRPC, and TCP traffic, along with fine-grained control of traffic behavior with rich routing rules, traffic encryption, in-depth telemetry and reporting, among other abilities. Istio is still under rapid/quickly development with different alpha versions of the API, so expect breaking changes going on. Also, plan for an extended ramp-up time to make productive use of Istio due to its complexity.

Source: [What is Istio?][istio-what]

You can easily deploy Istio on a cluster created via the Mesosphere Kubernetes Engine (MKE) or Konvoy. Just follow the guide below:

* [Setup Istio with MKE](setup-mke/)
* [Setup Istio with Konvoy](setup-konvoy/)
* [Demo: BookInfo](bookinfo/)
* [Demo: Intelligent Routing](routing/)
* [Demo: Security](security/)

[istio-what]: https://istio.io/docs/concepts/what-is-istio/

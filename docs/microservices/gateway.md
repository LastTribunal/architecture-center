# Implement an API gateway for microservices

An API gateway sits between clients and services, routing requests from clients to the right services. It may also perform various cross-cutting tasks such as monitoring, authentication, and SSL termination. This section describes some of reasons to use an API gateway, and considerations for implementing one.

![](./images/gateway.png)

In a microservices architecture, how should a client communicate with the various services that compose the application? One option is for services to expose public endpoints and have clients make direct HTTP calls to them. There are potential problems with this approach, however. 

- A single operation might require calls to multiple services. That can result in multiple network round trips between the client and the server, adding significant latency. 
- It can result in complex client code. The client must keep track of multiple endpoints, and handle failures in a resilient way. 
- It creates coupling between the client and the services that it calls, making it harder to refactor services.
- Each individual service must handle concerns such as authentication, SSL, and client rate limiting. 
- Services must expose a client-friendly protocol such as HTTP or WebSocket. This limits the choice of [communication protocols](./interservice-communication.md) 

To address these problems, an API gateway sits between the clients and the backend services. A gateway can perform a number of different functions. Depending on your scenario, you may not need all of them. These functions can be grouped into the following patterns:

[Gateway Routing](../patterns/gateway-routing.md). Route requests to one or more backend services, using layer 7 routing. The provides a single endpoint for clients, and helps to decouple clients from services. 

[Gateway Aggregation](../patterns/gateway-aggregation.md). Aggregate multiple individual requests into a single request. This pattern applies when a single operation requires calls to multiple backend services. The client sends one request to the gateway. The gateway dispatches requests to the various backend services, and then aggregates the results and sends them back to the client. This helps to reduce chattiness between the client and the backend. 

[Gateway Offloading](../patterns/gateway-offloading.md). Offload functionality from individual services to the gateway, particularly cross-cutting concerns. Some of the functions that a gateay can provide include SSL termination, authentication, certificate management, IP whitelisting, rate limiting, and logging and monitoring. It can be useful to consolidate these functions into one place, rather than making every service responsible for implementing them. This is paricularly true for features that requires specialized skills to implement correctly, such as authentication and authorization. 

Here are some of the main options for implementing an API gateway.

## Reverse proxy running in a container

Run Nginx, HAProxy, or another reverse proxy inside a container in your cluster. 

Because the reverse proxy runs inside a container, this approach is easy to configure and deploy. Nginx and HAProxy are both mature products with rich feature sets and high performance. You can extend them with third-party modules or by writing custom scripts in Lua. (Nginx also supports a JavaScript-based scripting module called NginScript.)

- To achieve high availability, deploy the gateway in a replica set for redundacy.  
- Use a ConfigMap to store the configuration file for the proxy, and mount the ConfigMap as a volume. 

An alternative way to deploy a reverse proxy in Kubernetes is by using an Ingress Controller. This lets you configure the reverse proxy as a Kubernetes resource. An Ingress Controller actually involves two separate Kubernetes resources, the *ingress* and the *ingress controller*.

- The Ingress resource defines the configuration for the reverse proxy, such as routing rules, TLS certificates, and...
- The Ingress Controller performs the reverse proxying. Several implementations exist, including Nginx and HAProxy. The Ingress Controller  watches the Ingress resource and updates the reverse proxy as needed.

One advantage of using an Ingress Controller is that it abstracts the ingress rules from the implementation details of the reverse proxy. You don't need to manage configuration files or container images. However, Ingress Controllers are still a beta feature of Kubernetes at the time of this writing, and will continue to evolve.

## Service mesh (Linkerd, Istio)

TBD

## Azure Application Gateway

Application Gateway is a managed load balancing service that can perform layer-7 routing and SSL termination. It also provides a web application firewall (WAF)

- Set the instance count to 2 or more for high availability.

To connect Application Gateway to a Kubernetes cluster in Azure:

1. Create an empty subnet in the cluster VNet.
2. Deploy Application Gateway.
3. Create a Kubernetes service with type=[NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport). This exposes the service on each node so that it can be reached frmo outside the cluster. It does not create a load balancer.
5. Get the assigned port number for the service.
6. Add an Application Gateway rule where:
    - The backend pool contains the agent VMs.
    - The HTTP setting specifies the service port number.
    - The gateway listener listens on ports 80/443
    

## Azure API Management 

API Management is a turnkey solution for publishing APIs to external and internal customers. It provides features that are useful for managing a public-facing API, including rate limiting, IP white listing, and authentication using Azure Active Directory.

To connect API Management to a Kubernetes cluster in Azure:

1. Create an empty subnet in the cluster VNet.
2. Deploy API Management to that subnet.
3. Create a Kubernetes service of type LoadBalancer. Use the [internal load balancer](https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer) annotation to create an internal load balancer, instead of an Internet-facing load balancer, which is the default.
4. Find the private IP of the internal load balancer, using kubectl or the Azure CLI.
5. Use API Management to create an API that directs to the private IP address of the load balancer.

If you use API Management, you should combine it with a reverse proxy, whether Nginx, HAProxy, or Azure Application Gateway.

> [!NOTE]
> To use API Management with Application Gateway, see [Integrate API Management in an internal VNET with Application Gateway](/azure/api-management/api-management-howto-integrate-internal-vnet-appgateway)






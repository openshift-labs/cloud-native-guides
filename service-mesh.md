## Microservice Tracing with Service Mesh

In this lab you will enable tracing and monitoring of your backend services using Service Mesh.

#### What is OpenShift Service Mesh?

Red Hat OpenShift Service Mesh is a platform that provides behavioral insight and operational control over the service mesh, providing a uniform way to connect, secure, and monitor microservice applications.

The term *service mesh* describes the network of microservices that make up applications in a distributed microservice architecture and the interactions between those microservices. As a service mesh grows in size and complexity, it can become harder to understand and manage.

Based on the open source [Istio](https://istio.io/) project, Red Hat OpenShift Service Mesh adds a transparent layer on existing distributed applications without requiring any changes to the service code. You add Red Hat OpenShift Service Mesh support to services by deploying a special sidecar proxy throughout your environment that intercepts all network communication between microservices. You configure and manage the service mesh using the control plane features.

Red Hat OpenShift Service Mesh provides an easy way to create a network of deployed services that provides discovery, load balancing, service-to-service authentication, failure recovery, metrics, and monitoring. A service mesh also provides more complex operational functionality, including A/B testing, canary releases, rate limiting, access control, and end-to-end authentication.

#### OpenShift Service Mesh Architecture

Red Hat OpenShift Service Mesh is logically split into a data plane and a control plane:

* The **data plane** is composed of a set of intelligent proxies deployed as sidecars. These proxies intercept and control all inbound and outbound network communication between microservices in the service mesh; sidecar proxies also communicate with Mixer, the general-purpose policy and telemetry hub.

* The **control plane** is responsible for managing and configuring proxies to route traffic, and configuring Mixers to enforce policies and collect telemetry.

![Service Mesh Architecture]({% image_path servicemesh-architecture.png %}){:width="400px"}

The components that make up the data plane and the control plane are:

* **Envoy proxy** - is the data plane component that intercepts all inbound and outbound traffic for all services in the service mesh. Envoy is deployed as a sidecar to the relevant service in the same pod.

* **Mixer** - is the control plane component responsible responsible for enforcing access control and usage policies (such as authorization, rate limits, quotas, authentication, request tracing) and collecting telemetry data from the Envoy proxy and other services.

* **Pilot** - is the control plane component responsible for configuring the proxies at runtime. Pilot provides service discovery for the Envoy sidecars, traffic management capabilities for intelligent routing (for example, A/B tests or canary deployments), and resiliency (timeouts, retries, and circuit breakers).

* **Citadel** - is the control plane component responsible for certificate issuance and rotation. Citadel provides strong service-to-service and end-user authentication with built-in identity and credential management. You can use Citadel to upgrade unencrypted traffic in the service mesh. Using Citadel, operators can enforce policies based on service identity rather than on network controls.

#### Enabling Service Mesh to Catalog Service

OpenShift Service Mesh automatically inject the sidecar into the Pod by specifying `sidecar.istio.io/inject:true` annotation in the DeploymentConfig.

In CodeReady Workspaces, click on Commands Palette and click on **SERVICE MESH > Catalog Service: Inject Istio Sidecar**

![Inject Sidecar]({% image_path  codeready-command-inject-catalog.png %}){:width="200px"}

To confirm that the application is successfully deployed, run this command:

~~~shell
$ oc get pods -lapp=catalog,deploymentconfig=catalog
NAME			READY	STATUS		RESTARTS	AGE
catalog-3-ppj47  	2/2	Running		1		8m
~~~

The status should be `Running` and there should be `2/2` pods in the `Ready` column.

#### Enabling Service Mesh to Inventory Service

Repeat the above step to enable Service Mesh to Inventory Service.
Click on Commands Palette and click on **SERVICE MESH > Inventory Service: Inject Istio Sidecar**

![Inject Sidecar]({% image_path  codeready-command-inject-inventory.png %}){:width="200px"}

To confirm that the application is successfully deployed, run this command:

~~~shell
$ oc get pods -lapp=inventory,deploymentconfig=inventory
NAME			READY	STATUS		RESTARTS	AGE
inventory-2-k6ftf	2/2	Running		1		3m
~~~

The status should be `Running` and there should be `2/2` pods in the `Ready` column.

#### Enabling Service Mesh to Gateway Service

Repeat the above step to enable Service Mesh to Gateway Service.
Click on Commands Palette and click on **SERVICE MESH > Gateway Service: Inject Istio Sidecar**

![Inject Sidecar]({% image_path  codeready-command-inject-gateway.png %}){:width="200px"}

To confirm that the application is successfully deployed, run this command:

~~~shell
$ oc get pods -lapp=gateway,deploymentconfig=gateway
NAME			READY	STATUS		RESTARTS	AGE
gateway-2-zqsmn		2/2	Running		1		1m
~~~

The status should be `Running` and there should be `2/2` pods in the `Ready` column.

#### Testing the application

To confirm that the Service Mesh is configured, run several times the following curl command:

~~~shell
$ curl -o /dev/null -s -w "%{http_code}\n" http://gateway.{{COOLSTORE_PROJECT}}.svc:8080/api/products
200
~~~

#### What is Kiali?

A Microservice Architecture breaks up the monolith into many smaller pieces that are composed together. Patterns to secure the communication between services like fault tolerance (via timeout, retry, circuit breaking, etc.) have come up as well as distributed tracing to be able to see where calls are going.

A service mesh can now provide these services on a platform level and frees the application writers from those tasks. Routing decisions are done at the mesh level.

[Kiali](https://www.kiali.io) works with Istio, in OpenShift or Kubernetes, to visualize the service mesh topology, to provide visibility into features like circuit breakers, request rates and more. It offers insights about the mesh components at different levels, from abstract Applications to Services and Workloads.

#### Observability with Kiali

Kiali provides an interactive graph view of your namespace in real time, being able to display the interactions at several levels (applications, versions, workloads), with contextual information and charts on the selected graph node or edge.

First, you need to access to Kiali. 
Launch a browser and navigate to [Kiali Console]({{ KIALI_URL }}) ({{ KIALI_URL }}). 
You should see the Kiali console login screen.

![Kiali- Log In]({% image_path kiali-login.png %}){:width="500px"}

Log in to the Kiali console as `{{OPENSHIFT_USER}}`/`{{OPENSHIFT_PASWORD}}`

After you log in, click on the `Graph` link in the left navigation and enter the following configuration:

 * Namespace: `{{COOLSTORE_PROJECT}}`
 * Display: check `Traffic Animation`
 * Fetching: `Last 5 min`

![Kiali- Graph]({% image_path kiali-graph.png %}){:width="700px"}

 This page shows a graph with all the microservices, connected by the requests going through them. On this page, you can see how the services interact with each other. 

#### Tracing with Kiali and Jaeger

[Kiali](https://www.kiali.io) includes [Jaeger Tracing](https://www.jaegertracing.io) to provide distributed tracing out of the box.
Jaeger, inspired by Dapper and OpenZipkin, is a distributed tracing system released as open source by Uber Technologies. It is used for monitoring and troubleshooting microservices-based distributed systems, including:

* Distributed context propagation
* Distributed transaction monitoring
* Root cause analysis
* Service dependency analysis
* Performance / latency optimization

From the [Kiali Console]({{ KIALI_URL }}) ({{ KIALI_URL }}), click on the `Distributed Tracing` link in the left navigation and enter the following configuration:

 * Select a Namespace: `{{COOLSTORE_PROJECT}}`
 * Select a Service: `gateway`
 * Then click on the **magnifying glass** on the right

![Kiali- Traces View]({% image_path kiali-traces-view.png %}){:width="700px"}

Let’s click on the trace title bar.

![Kiali- Trace Detail View]({% image_path kiali-trace-detail-view.png %}){:width="700px"}
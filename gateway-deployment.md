## Deploying the API Gateway

As explained in the introduction at the core of the Cool Store Portal architecture there is a Gateway component.

As we have done previously for the sake of saving time we're going to deploy our 'Gateway' service instead of coding it. But before we actually deploy the service we're going to explain briefly some concepts.

![CoolStore Architecture]({% image_path coolstore-arch-gateway-vertx.png %}){:width="400px"}

#### Technology

Our **'API Gateway'** service needed to be easy to scale, light and responsive, these are some of the reasons why we have implemented it as a Vert.x verticle.

#### What is Eclipse Vert.x?

[Eclipse Vert.x](http://vertx.io) is a toolkit for building reactive applications on the Java Virtual Machine (JVM). Vert.x does not impose a specific framework or packaging model and can be used within your existing applications and frameworks in order to add reactive functionality by just adding the Vert.x jar files to the application classpath.

Vert.x enables building reactive systems as defined by [The Reactive Manifesto](http://www.reactivemanifesto.org) and build 
services that are:

* *Responsive*: to handle requests in a reasonable time
* *Resilient*: to stay responsive in the face of failures
* *Elastic*: to stay responsive under various loads and be able to scale up and down
* *Message driven*: components interact using asynchronous message-passing

Vert.x is designed to be event-driven and non-blocking. Events are delivered in an event loop that must never be blocked. Unlike traditional applications, Vert.x uses a very small number of threads responsible for dispatching the events to event handlers. If the event loop is blocked, the events won’t be delivered anymore and therefore the code needs to be mindful of this execution model.

#### Vert.x Maven Project 

The `gateway-vertx` project has the following structure which shows the components of the Vert.x project laid out in different subdirectories according to Maven best practices:

![Gateway Project]({% image_path vertx-gateway-project.png %}){:width="340px"}

This is a minimal Vert.x project with support for RESTful services. This project currently contains no code other than the main class, `GatewayVerticle.java` which is there to bootstrap the Vert.x application. Verticles are encapsulated parts of the application that can run completely independently and communicate with each other via the built-in event bus in Vert.x. Verticles get deployed and run by Vert.x in an event loop and therefore it  is important that the code in a Verticle does not block. This asynchronous architecture allows Vert.x applications to easily scale and handle large amounts of throughput with few threads.All API calls in Vert.x by default are non-blocking and support this concurrency model.

![Vert.x Event Loop]({% image_path vertx-event-loop.png %}){:width="600px"}

Although you can have multiple, there is currently only one Verticle created in the `gateway-vertx` project. 

#### Deploying our API Gateway on OpenShift

It’s time to deploy our service on OpenShift. 

The API Gateway is using [Vert.x service discovery](http://vertx.io/docs/vertx-service-discovery/java) for finding where dependent services are deployed 
and accessing their endpoints. This service discovery can seamlessly integrated with external 
service discovery mechanisms provided by OpenShift, Kubernetes, Consul, Redis, etc.

[Vert.x service discovery](http://vertx.io/docs/vertx-service-discovery/java) integrates into OpenShift service discovery via OpenShift 
REST API and imports available services to make them available to the Vert.x application. Security 
in OpenShift comes first and therefore accessing the OpenShift REST API requires the user or the 
system (Vert.x in this case) to have sufficient permissions to do so. All containers in 
OpenShift run with a `serviceaccount` (by default, the project `default` service account) which can 
be used to grant permissions for operations like accessing the OpenShift REST API. You can read 
more about service accounts in the [OpenShift Documentation]({{OPENSHIFT_DOCS_BASE}}/dev_guide/service_accounts.html) and this 
[blog post](https://blog.openshift.com/understanding-service-accounts-sccs/#_service_accounts)

Grant permission to the API Gateway to be able to access OpenShift REST API and discover services.

> Make sure to replace the project name with your own unique project name

~~~shell
$ oc policy add-role-to-user view -n {{COOLSTORE_PROJECT}} -z default
~~~ 

OpenShift [Source-to-Image (S2I)]({{OPENSHIFT_DOCS_BASE}}/architecture/core_concepts/builds_and_image_streams.html#source-build) 
feature can be used to build a container image from a git repository. OpenShift S2I uses the [supported OpenJDK container image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift) to build the final container image of the 
Inventory service by building the WildFly Swam uber-jar from source code (build strategy **'Source'**), using Maven, to the OpenShift platform.

Vert.x service discovery integrates into OpenShift service discovery via OpenShift REST API and imports available services to make them available to the Vert.x application. Security in OpenShift comes first and therefore accessing the OpenShift REST API requires the user or the system (Vert.x in this case) to have sufficient permissions to do so. All containers in OpenShift run with a `serviceaccount` (by default, the project `default` service account) which can be used to grant permissions for operations like accessing the OpenShift REST API. You can read more about service accounts in the [OpenShift Documentation]({{OPENSHIFT_DOCS_BASE}}/dev_guide/service_accounts.html) and this [blog post](https://blog.openshift.com/understanding-service-accounts-sccs/#_service_accounts)

Next commands are going to deploy our API Gateway service.

* **Name:** gateway
* **S2I runtime:** redhat-openjdk18-openshift
* **Image tag:** 1.4
* **Repository:** {{LABS_GIT_REPO}}
* **Context Directory:** gateway-vertx

~~~shell
$ oc new-app redhat-openjdk18-openshift:1.4~{{LABS_GIT_REPO}} \
        --context-dir=gateway-vertx \
        --name=gateway

$ oc expose svc/gateway
~~~

Once this completes, your project should be up and running. You can see the expose DNS url for the Gateway service in the OpenShift Web Console or using OpenShift CLI.

~~~shell
$ oc get routes

NAME        HOST/PORT                                                  PATH      SERVICES    PORT       TERMINATION   
catalog     catalog-{{COOLSTORE_PROJECT}}-{{OPENSHIFT_USER}}.roadshow.openshiftapps.com               catalog     8080                     None
inventory   inventory-{{COOLSTORE_PROJECT}}-{{OPENSHIFT_USER}}.roadshow.openshiftapps.com             inventory   8080                     None
gateway     gateway-{{COOLSTORE_PROJECT}}-{{OPENSHIFT_USER}}.roadshow.openshiftapps.com               gateway     8080                     None
~~~

Copy the route url for API Gateway and verify the API Gateway service works using `curl`:

> The route urls in your project would be different from the ones in this lab guide! Use the ones from your project.

~~~shell
$ curl http://{{API_GATEWAY_ROUTE_HOST}}/api/products

[ {
  "itemId" : "329299",
  "name" : "Red Fedora",
  "desc" : "Official Red Hat Fedora",
  "price" : 34.99,
  "availability" : {
    "quantity" : 35
  }
},
...
]
~~~

As mentioned earlier, Vert.x built-in service discovery is integrated with OpenShift service discovery to lookup the Catalog and Inventory APIs.

Well done! You are ready to move on to the next lab.

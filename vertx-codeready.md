## Reactive Microservices with Eclipse Vert.x

In this lab you will learn about Eclipse Vert.x and how you can build microservices using reactive principles. During this lab you will create a scalable API Gateway that aggregates gateway and Inventory APIs.

![CoolStore Architecture]({% image_path coolstore-arch-gateway.png %}){:width="500px"}

#### What is Eclipse Vert.x?

[Eclipse Vert.x](http://vertx.io) is a toolkit for building reactive applications on the Java Virtual Machine (JVM). Vert.x does not 
impose a specific framework or packaging model and can be used within your existing applications and frameworks 
in order to add reactive functionality by just adding the Vert.x jar files to the application classpath.

Vert.x enables building reactive systems as defined by [The Reactive Manifesto](http://www.reactivemanifesto.org) and build 
services that are:

* *Responsive*: to handle requests in a reasonable time
* *Resilient*: to stay responsive in the face of failures
* *Elastic*: to stay responsive under various loads and be able to scale up and down
* *Message driven*: components interact using asynchronous message-passing

Vert.x is designed to be event-driven and non-blocking. Events are delivered in an event loop that must never be blocked. Unlike traditional applications, Vert.x uses a very small number of threads responsible for dispatching the events to event handlers. If the event loop is blocked, the events won’t be delivered anymore and therefore the code needs to be mindful of this execution model.

#### Vert.x Maven Project 

The `gateway-vertx` project has the following structure which shows the components of 
the Vert.x project laid out in different subdirectories according to Maven best practices:

![Gateway Project]({% image_path vertx-gateway-project.png %}){:width="200px"}

This is a minimal Vert.x project with support for RESTful services. This project currently contains no code
other than the main class, `GatewayVerticle.java` which is there to bootstrap the Vert.x application. Verticles
are encapsulated parts of the application that can run completely independently and communicate with each other
via the built-in event bus in Vert.x. Verticles get deployed and run by Vert.x in an event loop and therefore it 
is important that the code in a Verticle does not block. This asynchronous architecture allows Vert.x applications 
to easily scale and handle large amounts of throughput with few threads.All API calls in Vert.x by default are non-blocking 
and support this concurrency model.

![Vert.x Event Loop]({% image_path vertx-event-loop.png %}){:width="600px"}

Although you can have multiple, there is currently only one Verticle created in the `gateway-vertx` project. 

Examine `GatewayVerticle` class in the `com.redhat.cloudnative.gateway` package in the `src` directory.

~~~java
package com.redhat.cloudnative.gateway;


import io.vertx.core.AbstractVerticle;
import io.vertx.core.Future;
import io.vertx.ext.web.Router;
import io.vertx.ext.web.handler.StaticHandler;

public class GatewayVerticle extends AbstractVerticle {
    @Override
    public void start(Future<Void> future) {
        Router router = Router.router(vertx);

        router.get("/").handler(StaticHandler.create("assets"));

        vertx.createHttpServer().requestHandler(router::accept)
            .listen(Integer.getInteger("http.port", 8080));
    }
}
~~~

Here is what happens in the above code:

1. A Verticle is created by extending from `AbstractVerticle` class
2. `Router` is retrieved for mapping the REST endpoints
3. A REST endpoint is created for `/` to return a static HTML page `assets/index.html`
4. An HTTP Server is created which listens on port 8080

#### Creating an Openshift Application

An application is an umbrella of components that work together to implement the overall application. OpenShift helps organize these modular applications with a concept called, appropriately enough, the application. An OpenShift application represents all of an app's components in a logical management unit.

First, create an application called `gateway` to work with:

~~~shell
$ odo app create gateway
Creating application: gateway in project: {{COOLSTORE_PROJECT}}
Switched to application: gateway in project: {{COOLSTORE_PROJECT}}
~~~

You can verify that the new application is created with the following commands:

~~~shell
$ odo app list
The project '{{COOLSTORE_PROJECT}}' has the following applications:
ACTIVE     NAME
           inventory
           catalog
*          gateway
~~~

#### Creating a Service Component from Binary

You can use Maven to make sure the skeleton project builds successfully. You should get a `BUILD SUCCESS` message 
in the build logs, otherwise the build has failed.

In Eclipse Che, click on **gateway-vertx** project in the project explorer, 
and then click on Commands Palette and click on **BUILD > build**.

![Maven Build]({% image_path codeready-commands-build.png %}){:width="200px"}

Once successfully built, the resulting `jar` is located in the `target/` directory:


~~~shell
$ ls labs/gateway-vertx/target/*.jar

labs/gateway-vertx/target/gateway-1.0-SNAPSHOT.jar
~~~

This is an uber-jar with all the dependencies required packaged in the *jar* to enable running the 
application with `java -jar`.

Now, add a component named `service` of type `redhat-openjdk18-openshift:1.4` to the application `gateway` and deploy the uber-jar `gateway-1.0-SNAPSHOT.jar`:

~~~shell
$ odo create redhat-openjdk18-openshift:1.4 service --app gateway \
--binary labs/gateway-vertx/target/gateway-1.0-SNAPSHOT.jar \
--env JAVA_OPTIONS="-Dcatalog.api.host=service-catalog -Dcatalog.api.port=8080 -Dinventory.api.host=service-inventory -Dinventory.api.port=8080"
 ✓   Checking component
 ✓   Checking component version
 ✓   Creating component service
 OK  Component 'service' was created and ports 8080/TCP,8443/TCP,8778/TCP were opened
 OK  Component 'service' is now set as active component
To push source code to the component run 'odo push'
~~~

![gateway Service Component]({% image_path vertx-gateway-component.png %}){:width="500px"}

#### Pushing your source code

Now that the component is running, push our initial source code:

~~~shell
$ odo push service --app gateway
Pushing changes to component: service
 ✓   Waiting for pod to start
 ✓   Copying files to pod
 ✓   Building component
 OK  Changes successfully pushed to component: service
~~~

The jar file has now been pushed to the container, and the process in that container restarted.

#### Creating an OpenShift route

To access to the service outside the cluster, create an external URL (an Openshift Route) for the `gateway` application:

~~~shell
$ odo url create --app gateway --component service --port 8080
Adding URL to component: service
 OK  URL created for component: service

service - http://service-gateway-{{COOLSTORE_PROJECT}}.apps.openshiftworkshop.com
~~~

> The route urls in your project would be different from the ones in this lab guide! Use the ones from your project.

Copy the route url for the gateway service into your browser:

![gateway Service Root]({% image_path vertx-gateway-service-root.png %}){:width="500s"}

#### Updating Component on Change

Watch for Changes and updating Component on Change. Let's run `odo watch ` in a new terminal window.

~~~shell
$ odo watch service --app gateway
Waiting for something to change in /projects/labs/gateway-vertx/target/gateway-1.0-SNAPSHOT.jar
~~~

#### Creating an API Gateway

In the previous labs, you have created two RESTful services: gateway and Inventory. Instead of the 
web frontend contacting each of these backend services, you can create an API Gateway which is an entry 
point for for the web frontend to access all backend services from a single place. This pattern is expectedly 
called [API Gateway](http://microservices.io/patterns/apigateway.html) and is a common practice in Microservices 
architecture.

Replace the content of `src/main/java/com/redhat/cloudnative/gateway/GatewayVerticle.java` class with the following:

~~~java
package com.redhat.cloudnative.gateway;

import io.vertx.core.http.HttpMethod;
import io.vertx.core.json.Json;
import io.vertx.core.json.JsonObject;
import io.vertx.ext.web.client.WebClientOptions;
import io.vertx.rxjava.core.AbstractVerticle;
import io.vertx.rxjava.ext.web.Router;
import io.vertx.rxjava.ext.web.RoutingContext;
import io.vertx.rxjava.ext.web.client.WebClient;
import io.vertx.rxjava.ext.web.codec.BodyCodec;
import io.vertx.rxjava.ext.web.handler.CorsHandler;
import io.vertx.rxjava.ext.web.handler.StaticHandler;
import io.vertx.rxjava.servicediscovery.ServiceDiscovery;
import io.vertx.rxjava.servicediscovery.types.HttpEndpoint;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import rx.Observable;
import rx.Single;

public class GatewayVerticle extends AbstractVerticle {
    private static final Logger LOG = LoggerFactory.getLogger(GatewayVerticle.class);

    private WebClient catalog;
    private WebClient inventory;

    @Override
    public void start() {
        Router router = Router.router(vertx);
        router.route().handler(CorsHandler.create("*").allowedMethod(HttpMethod.GET));
        router.get("/").handler(StaticHandler.create("assets"));
        router.get("/health").handler(ctx -> ctx.response().end(new JsonObject().put("status", "UP").toString()));
        router.get("/api/products").handler(this::products);

        ServiceDiscovery.create(vertx, discovery -> {
            // Catalog lookup
            Single<WebClient> catalogDiscoveryRequest = HttpEndpoint.rxGetWebClient(discovery,
                    rec -> rec.getName().equals("catalog"))
                    .onErrorReturn(t -> WebClient.create(vertx, new WebClientOptions()
                            .setDefaultHost(System.getProperty("catalog.api.host", "localhost"))
                            .setDefaultPort(Integer.getInteger("catalog.api.port", 9000))));

            // Inventory lookup
            Single<WebClient> inventoryDiscoveryRequest = HttpEndpoint.rxGetWebClient(discovery,
                    rec -> rec.getName().equals("inventory"))
                    .onErrorReturn(t -> WebClient.create(vertx, new WebClientOptions()
                            .setDefaultHost(System.getProperty("inventory.api.host", "localhost"))
                            .setDefaultPort(Integer.getInteger("inventory.api.port", 9001))));

            // Zip all 3 requests
            Single.zip(catalogDiscoveryRequest, inventoryDiscoveryRequest, (c, i) -> {
                // When everything is done
                catalog = c;
                inventory = i;
                return vertx.createHttpServer()
                    .requestHandler(router::accept)
                    .listen(Integer.getInteger("http.port", 8080));
            }).subscribe();
        });
    }

    private void products(RoutingContext rc) {
        // Retrieve catalog
        catalog.get("/api/catalog").as(BodyCodec.jsonArray()).rxSend()
            .map(resp -> {
                if (resp.statusCode() != 200) {
                    new RuntimeException("Invalid response from the catalog: " + resp.statusCode());
                }
                return resp.body();
            })
            .flatMap(products ->
                // For each item from the catalog, invoke the inventory service
                Observable.from(products)
                    .cast(JsonObject.class)
                    .flatMapSingle(product ->
                        inventory.get("/api/inventory/" + product.getString("itemId")).as(BodyCodec.jsonObject())
                            .rxSend()
                            .map(resp -> {
                                if (resp.statusCode() != 200) {
                                    LOG.warn("Inventory error for {}: status code {}",
                                            product.getString("itemId"), resp.statusCode());
                                    return product.copy();
                                }
                                return product.copy().put("availability", 
                                    new JsonObject().put("quantity", resp.body().getInteger("quantity")));
                            }))
                    .toList().toSingle()
            )
            .subscribe(
                list -> rc.response().end(Json.encodePrettily(list)),
                error -> rc.response().end(new JsonObject().put("error", error.getMessage()).toString())
            );
    }
}
~~~

Let's break down what happens in the above code. The `start` method creates an HTTP 
server and a REST mapping to map `/api/products` to the `products` Java 
method. 

Vert.x provides [built-in service discovery](http://vertx.io/docs/vertx-service-discovery/java) 
for finding where dependent services are deployed 
and accessing their endpoints. Vert.x service discovery can be seamlessly integrated with external 
service discovery mechanisms provided by OpenShift, Kubernetes, Consul, Redis, etc.

In this lab, since you will deploy the API Gateway on OpenShift, the OpenShift service discovery 
bridge is used to automatically import OpenShift services into the Vert.x application as they 
get deployed and undeployed. Since you also want to test the API Gateway locally, there is an 
`onErrorReturn()` clause in the service lookup to fallback on a local service for Inventory 
and gateway REST APIs. 


The `products` method invokes the gateway REST endpoint and retrieves the products. It then 
iterates over the retrieved products and for each product invokes the 
Inventory REST endpoint to get the inventory status and enrich the product data with availability 
info. 

Note that instead of making blocking calls to the gateway and Inventory REST APIs, all calls 
are non-blocking and handled using [RxJava](http://vertx.io/docs/vertx-rx/java). Due to its non-blocking 
nature, the `product` method can immediately return without waiting for the gateway and Inventory 
REST invocations to complete and whenever the result of the REST calls is ready, the result 
will be acted upon and update the response which is then sent back to the client.

Build and package the Gateway service using Maven by clicking on **BUILD > build** from the commands palette.

![Maven Build]({% image_path codeready-commands-build.png %}){:width="200px"}

Once successfully built, your new version of the jar will be pushed automatically into the Gateway Component thanks to the `odo watch` command. You should see following logs in the Terminal where you ran the `odo watch` command.

~~~shell
File /projects/labs/gateway-vertx/target/gateway-1.0-SNAPSHOT.jar changed
Pushing files...
 ✓   Waiting for pod to start
 ✓   Copying files to pod
 ✓   Building component
Waiting for something to change in /projects/labs/gateway-vertx/target/gateway-1.0-SNAPSHOT.jar
~~~

Now, you can access the Catalog REST API. Let’s test it out using `curl` in a new terminal window:

~~~shell
$ odo url list --component service --app gateway
NAME        URL                                                              PORT
service     http://service-gateway-{{COOLSTORE_PROJECT}}.apps.openshiftworkshop.com      8080

$ curl http://service-gateway-{{COOLSTORE_PROJECT}}.apps.openshiftworkshop.com/api/products
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

Well done! You are ready to move on to the next lab.

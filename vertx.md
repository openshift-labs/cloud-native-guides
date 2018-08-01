## Reactive Microservices with Eclipse Vert.x

In this lab you will learn about Eclipse Vert.x and how you can 
build microservices using reactive principles. During this lab you will 
create a scalable API Gateway that aggregates Catalog and Inventory APIs.


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

~~~shell
├── pom.xml               # The Maven project file
└── src
    └── main
        └── java          # The source code to the project
        └── resources     # The static resource files and configurations
~~~

This is a minimal Vert.x project with support for RESTful services. This project currently contains no code
other than the main class, `GatewayVerticle.java` which is there to bootstrap the Vert.x application. Verticles
are encapsulated parts of the application that can run completely independently and communicate with each other
via the built-in event bus in Vert.x. Verticles get deployed and run by Vert.x in an event loop and therefore it 
is important that the code in a Verticle does not block. This asynchronous architecture allows Vert.x applications 
to easily scale and handle large amounts of throughput with few threads.All API calls in Vert.x by default are non-blocking 
and support this concurrency model.

![Vert.x Event Loop]({% image_path vertx-event-loop.png %}){:width="600px"}

Although you can have multiple, there is currently only one Verticle created in the `gateway-vertx` project. 

Examine `src/main/java/com/redhat/cloudnative/gateway/GatewayVerticle.java`

~~~java
import io.vertx.core.AbstractVerticle;
import io.vertx.core.Future;
import io.vertx.ext.web.Router;

public class GatewayVerticle extends AbstractVerticle {
    @Override
    public void start(Future<Void> future) {
        Router router = Router.router(vertx);

        router.get("/*").handler(rc -> {
            rc.response().end("{\"message\": \"Hello World\"}");
        });

        vertx.createHttpServer().requestHandler(router::accept)
            .listen(Integer.getInteger("http.port", 8080));
    }
}
~~~

Here is what happens in the above code:

1. A Verticle is created by extending from `AbstractVerticle` class
2. `Router` is retrieved for mapping the REST endpoints
3. A REST endpoint is created for `/*` to return a static JSON response `{"message": "Hello World"}`
3. An HTTP Server is created which listens on port 8080

You can use Maven to make sure the skeleton project builds successfully. You should get a `BUILD SUCCESS` message 
in the build logs, otherwise the build has failed.

> Make sure to run the `package` Maven goal and not `install`. The latter would 
> download a lot more dependencies and do things you don't need yet!

~~~shell
$ cd gateway-vertx
$ mvn package

...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 2.769 s
[INFO] Finished at: 2017-07-28T11:44:11+07:00
[INFO] Final Memory: 20M/308M
[INFO] ------------------------------------------------------------------------
~~~

Once built, the resulting *jar* is located in the `target/` directory:

~~~shell
$ ls target/*.jar

target/gateway-1.0-SNAPSHOT.jar
~~~

This is an uber-jar with all the dependencies required packaged in the *jar* to enable running the 
application with `java -jar`.

You can run the Vert.x application using `java -jar` or conveniently using `vertx:run` goal from 
the `vertx-maven-plugin`:

~~~shell
$ mvn vertx:run

...
[INFO] Succeeded in deploying verticle
...
~~~

Verify the application is working using `curl` in a new terminal window:

~~~shell
$ curl http://localhost:8080

{"message": "Hello World"}
~~~

Note that while the application is running using `mvn vertx:run`, you can make changes in the code
and they would immediately be compiled and updated in the running application to provide fast
feedback to the developer.

Now that the project is ready, let's get coding!

#### Create an API Gateway

In the previous labs, you have created two RESTful services: Catalog and Inventory. Instead of the 
web front contacting each of these backend services, you can create an API Gateway which is an entry 
point for for the web front to access all backend services from a single place. This pattern is expectedly 
called [API Gateway](http://microservices.io/patterns/apigateway.html) and is a common practice in Microservices 
architecture.

![API Gateway Pattern]({% image_path coolstore-arch.png %}){:width="400px"}

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
and accessing their endpoints. Vert.x service discovery can seamlessly integrated with external 
service discovery mechanisms provided by OpenShift, Kubernetes, Consul, Redis, etc.

In this lab, since you will deploy the API Gateway on OpenShift, the OpenShift service discovery 
bridge is used to automatically import OpenShift services into the Vert.x application as they 
get deployed and undeployed. Since you also want to test the API Gateway locally, there is an 
`onErrorReturn()` clause in the service lookup to fallback on a local service for Inventory 
and Catalog REST APIs. 

~~~java
public void start() {
    Router router = Router.router(vertx);
    router.route().handler(CorsHandler.create("*").allowedMethod(HttpMethod.GET));
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
~~~

The `products` method invokes the Catalog REST endpoint and retrieves the products. It then 
iterates over the retrieved products and for each product invokes the 
Inventory REST endpoint to get the inventory status and enrich the product data with availability 
info. 

Note that instead of making blocking calls to the Catalog and Inventory REST APIs, all calls 
are non-blocking and handled using [RxJava](http://vertx.io/docs/vertx-rx/java). Due to its non-blocking 
nature, the `product` method can immediately return without waiting for the Catalog and Inventory 
REST invocations to complete and whenever the result of the REST calls is ready, the result 
will be acted upon and update the response which is then sent back to the client.

~~~java
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
~~~

Run the maven build to make sure the code compiles successfully.

~~~shell
$ mvn package
~~~

Since the API Gateway requires the Catalog and Inventory services to be running, let's run all three 
services simultaneously and verify that the API Gateway works as expected. 

Open a new terminal window and start the Catalog service:

~~~shell
$ cd catalog-spring-boot
$ mvn spring-boot:run
~~~

Open another new terminal window and start the Inventory service:

~~~shell
$ cd inventory-wildfly-swarm
$ mvn wildfly-swarm:run
~~~

Now that Catalog and Inventory services are up and running, start the API Gateway service in a new terminal window:

~~~shell
$ cd gateway-vertx
$ mvn vertx:run 
~~~

> You will see the following exception in the logs: `java.io.FileNotFoundException: /.../kubernetes.io/serviceaccount/token`
> 
> This is expected and is the result of Vert.x trying to import services form OpenShift. Since you are 
> running the API Gateway on your local machine, the lookup fails and falls back to the local service 
> lookup. It's all good!

Now you can test the API Gateway by hitting the `/api/products` endpoint using `curl`:

~~~shell
$ curl http://localhost:8080/api/products

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

Note that the JSON response aggregates responses from Catalog and Inventory services and 
the inventory info for each product is available within the same JSON object.

Stop all services by pressing `CTRL-C` in the terminal windows.

#### Deploy Vert.x on OpenShift

It’s time to build and deploy our service on OpenShift. 

Like discussed, Vert.x service discovery integrates into OpenShift service discovery via OpenShift 
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
feature can be used to build a container image from your project. OpenShift 
S2I uses the [supported OpenJDK container image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift) to build the final container 
image of the API Gateway service by uploading the Vert.x uber-jar from 
the `target` folder to the OpenShift platform. 

Maven projects can use the [Fabric8 Maven Plugin](https://maven.fabric8.io) in order to use OpenShift S2I for building 
the container image of the application from within the project. This maven plugin is a Kubernetes/OpenShift client 
able to communicate with the OpenShift platform using the REST endpoints in order to issue the commands 
allowing to build a project, deploy it and finally launch a docker process as a pod.

To build and deploy the Inventory service on OpenShift using the `fabric8` maven plugin, run the following Maven command:

~~~shell
$ mvn fabric8:deploy
~~~

During the deployment, you might see that Fabric8 Maven Plugin throws an `java.util.concurrent.RejectedExecutionException` 
exception. This is due to [a bug](https://github.com/fabric8io/kubernetes-client/issues/1035) in one of Fabric8 Maven Plugin 
dependencies which is being worked on right now and will be fixed soon. You can ignore this exception for now. The deployment 
nevertheless succeeds.

`fabric8:deploy` will cause the following to happen:

* The API Gateway uber-jar is built using Vert.x
* A container image is built on OpenShift containing the API Gateway uber-jar and JDK
* All necessary objects are created within the OpenShift project to deploy the API Gateway service

Once this completes, your project should be up and running. OpenShift runs the different components of 
the project in one or more pods which are the unit of runtime deployment and consists of the running 
containers for the project. 

Let's take a moment and review the OpenShift resources that are created for the API Gateway:

* **Build Config**: `gateway-s2i` build config is the configuration for building the Gateway 
container image from the gateway source code or JAR archive
* **Image Stream**: `gateway` image stream is the virtual view of all gateway container 
images built and pushed to the OpenShift integrated registry.
* **Deployment Config**: `gateway` deployment config deploys and redeploys the Gateway container 
image whenever a new Gateway container image becomes available
* **Service**: `gateway` service is an internal load balancer which identifies a set of 
pods (containers) in order to proxy the connections it receives to them. Backing pods can be 
added to or removed from a service arbitrarily while the service remains consistently available, 
enabling anything that depends on the service to refer to it at a consistent address (service name 
or IP).
* **Route**: `gateway` route registers the service on the built-in external load-balancer 
and assigns a public DNS name to it so that it can be reached from outside OpenShift cluster.

You can review the above resources in the OpenShift Web Console or using `oc describe` command:

> `bc` is the short-form of `buildconfig` and can be interchangeably used instead of it with the
> OpenShift CLI. The same goes for `is` instead of `imagestream`, `dc` instead of`deploymentconfig` 
> and `svc` instead of `service`.

~~~shell
$ oc describe bc gateway-s2i
$ oc describe is gateway
$ oc describe dc gateway
$ oc describe svc gateway
$ oc describe route gateway
~~~

You can see the expose DNS url for the Gateway service in the OpenShift Web Console or using 
OpenShift CLI.

~~~shell
$ oc get routes

NAME        HOST/PORT                                                  PATH      SERVICES    PORT       TERMINATION   
catalog     catalog-{{COOLSTORE_PROJECT}}.roadshow.openshiftapps.com               catalog     8080                     None
inventory   inventory-{{COOLSTORE_PROJECT}}.roadshow.openshiftapps.com             inventory   8080                     None
gateway     gateway-{{COOLSTORE_PROJECT}}.roadshow.openshiftapps.com               gateway     8080                     None
~~~

Copy the route url for API Gateway and verify the API Gateway service works using `curl`:

> The route urls in your project would be different from the ones in this lab guide! Use the ones from yor project.

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

As mentioned earlier, Vert.x built-in service discovery is integrated with OpenShift service 
discovery to lookup the Catalog and Inventory APIs.

Well done! You are ready to move on to the next lab.

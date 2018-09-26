## Service Resilience and Fault Tolerance

In this lab you will learn about how you can build service resilience and fault-tolerant into 
the applications both at the infrastructure level using OpenShift capabilities as well as 
at the application level using circuit breakers to prevent cascading failures when 
downstream dependencies fail.

#### Scaling Up Applications

Applications capacity for serving clients is bounded by the amount of computing power 
allocated to them and although it's possible to increase the computing power per instance, 
it's far easier to keep the application instances within reasonable sizes and 
instead add more instances to increase serving capacity. Traditionally, due to 
the stateful nature of most monolithic applications, increasing capacity had been achieved 
via scaling up the application server and the underlying virtual or physical machine by adding 
more cpu and memory (vertical scaling). Cloud-native apps however are stateless and can be 
easily scaled up by spinning up more application instances and load-balancing requests 
between those instances (horizontal scaling).

![Scaling Up vs Scaling Out]({% image_path fault-scale-up-vs-out.png %}){:width="500px"}

In previous labs, you learned how to build container images from your application code and 
deploy them on OpenShift. Container images on OpenShift follow the 
[immutable server](https://martinfowler.com/bliki/ImmutableServer.html) pattern which guarantees 
your application instances will always starts from a known well-configured state and makes 
deploying instances a repeatable practice. Immutable server pattern simplifies scaling out 
application instances to starting a new instance which is guaranteed to be identical to the 
existing instances and adding it to the load-balancer.

Now, let's use the `oc scale` command to scale up the Web UI pod in the CoolStore retail 
application to 2 instances. In OpenShift, deployment config is responsible for starting the 
application pods and ensuring the specified number of instances for each application pod 
is running. Therefore the number of pods you want to scale to should be defined on the 
deployment config.

> You can scale pods up and down via the OpenShift Web Console by clicking on the up and 
> down arrows on the right side of each pods blue circle.

First, get list of deployment configs available in the project.

~~~shell
$ oc project {{COOLSTORE_PROJECT}}
$ oc get dc 

NAME        REVISION   DESIRED   CURRENT   TRIGGERED BY
catalog     1          1         1         config,image(catalog:latest)
gateway     1          1         1         config,image(gateway:latest)
inventory   1          1         1         config,image(inventory:latest)
web         1          1         1         config,image(web:latest)
~~~

And then, scale the `web` deployment config to 2 pods:

~~~shell
$ oc scale dc/web --replicas=2
~~~

The `--replicas` option specified the number of Web UI pods that should be running. If you look 
at the OpenShift Web Console, you can see a new pod is being started for the Web UI and as soon 
as the health probes pass, it will be automatically added to the load-balancer.

![Scaling Up Pods]({% image_path fault-scale-up.png %}){:width="740px"}

You can verify that the new pod is added to the load balancer by checking the details of the 
Web UI service object:

~~~shell
$ oc describe svc/web

...
Endpoints:              10.129.0.146:8080,10.129.0.232:8080
...
~~~

`Endpoints` shows the IPs of the 2 pods that the load-balancer is sending traffic to.

> The load-balancer by default, sends the client to the same pod on consequent requests. The 
> [load-balancing strategy](https://docs.openshift.com/container-platform/3.5/architecture/core_concepts/routes.html#load-balancing) 
> can be specified using an annotation on the route object. Run the following to change the load-balancing 
> strategy to round robin: 
> 
>     $ oc annotate route/web haproxy.router.openshift.io/balance=roundrobin
>

#### Scaling Applications on Auto-pilot

Although scaling up and scaling down pods are automated and easy using OpenShift, however it still 
requires a person or a system to run a command or invoke an API call (to OpenShift REST API. Yup! there
is a REST API for all OpenShift operations) to scale the applications. That in turn needs to be in response 
to some sort of increase to the application load and therefore the person or the system needs to be aware of 
how much load the application is handling at all times to make the scaling decision.

OpenShift automates this aspect of scaling as well via automatically scaling the application pods up 
and down within a specified min and max boundary based on the container metrics such as cpu and memory 
consumption. In that case, if there is a surge of users visiting the CoolStore online shop due to 
holiday season coming up or a good deal on a product, OpenShift would automatically add more pods to 
handle the increase load on the application and after the load goes, the application is automatically 
scaled down to free up compute resources.

In order to define auto-scaling for a pod, we should first define how much cpu and memory a pod is 
allowed to consume which will act as a guideline for OpenShift to know when to scale the pod up or 
down. Since the deployment config is used when starting the application pods, the application pod resource 
(cpu and memory) containers should also be defined on the deployment config.

When allocating compute resources to application pods, each container may specify a *request*
and a *limit*value each for CPU and memory. The 
[*request*]({{OPENSHIFT_DOCS_BASE}}/dev_guide/compute_resources.html#dev-memory-requests) 
values define how much resource should be dedicated to an application pod so that it can run. It's 
the minimum resources needed in other words. The 
[*limit*]({{OPENSHIFT_DOCS_BASE}}/dev_guide/compute_resources.html#dev-memory-limits) values 
defines how much resource an application pod is allowed to consume, if there is more resources 
on the node available than what the pod has requested. This is to allow various quality of service 
tiers with regards to compute resources. You can read more about these quality of service tiers 
in [OpenShift Documentation]({{OPENSHIFT_DOCS_BASE}}/dev_guide/compute_resources.html#quality-of-service-tiers).

Set the following resource constraints on the Web UI pod:

* Memory Request: 256 MB
* Memory Limit: 512 MB
* CPU Request: 200 millicore
* CPU Limit: 300 millicore

> CPU is measured in units called millicores. Each node in a cluster inspects the 
> operating system to determine the amount of CPU cores on the node, then multiplies 
> that value by 1000 to express its total capacity. For example, if a node has 2 cores, 
> the node’s CPU capacity would be represented as 2000m. If you wanted to use 1/10 of 
> a single core, it would be represented as 100m. Memory is measured in 
> bytes and is specified with [SI suffices]({{OPENSHIFT_DOCS_BASE}}/dev_guide/compute_resources.html#dev-compute-resources) 
> (E, P, T, G, M, K) or their power-of-two-equivalents (Ei, Pi, Ti, Gi, Mi, Ki).

~~~shell
$ oc set resources dc/web --limits=cpu=400m,memory=512Mi --requests=cpu=200m,memory=256Mi

deploymentconfig "web" resource requirements updated
~~~

> You can also use the OpenShift Web Console by clicking on **Applications** >> **Deployments** within 
> the **{{COOLSTORE_PROJECT}}** project. Click then on **web** and from the **Actions** menu on 
> the top-right, choose **Edit Resource Limits**.

The pods get restarted automatically setting the new resource limits in effect. Now you can define an 
autoscaler using `oc autoscale` command to scale the Web UI pods up to 5 instances whenever 
the CPU consumption passes 50% utilization:

> You can configure an autoscaler using OpenShift Web Console by clicking 
> on **Applications** >> **Deployments** within 
> the **{{COOLSTORE_PROJECT}}** project. Click then on **web** and from the **Actions** menu on 
> the top-right, choose **Edit Autoscaler**.

~~~shell
$ oc autoscale dc/web --min 1 --max 5 --cpu-percent=40

deploymentconfig "web" autoscaled
~~~

All set! Now the Web UI can scale automatically to multiple instances if the load on the CoolStore 
online store increases. You can verify that using for example the `siege` command-line utility, which 
is a handy tool for running load tests against web endpoints and is already 
installed within your Eclipse Che workspace. 

Run the following command in the **Terminal** window.
~~~shell
$ siege -c80 -d2 -t5M http://web.coolstore-XX.svc.cluster.local:8080
~~~

Note that you are using the internal url of the Web UI in this command. Since Eclipse Che is running on 
the same OpenShift cluster as Web UI, you can choose to use the external URL that is exposed on the load balancer 
or the internal user which goes directly to the Web UI pod and bypasses the load balancer. You can 
read more about internal service dns names in 
[OpenShift Docs]({{OPENSHIFT_DOCS_BASE}}/architecture/networking/networking.html).

As the load is generated, you will notice that it will create a spike in the 
Web UI cpu usage and trigger the autoscaler to scale the Web UI container to 5 pods (as configured 
on the deployment config) to cope with the load.

> Depending on the resources available on the OpenShift cluster in the lab environment, 
> the Web UI might scale to fewer than 5 pods to handle the extra load. Run the command again 
> to generate more load.

![Web UI Automatically Scaled]({% image_path fault-autoscale-web.gif %}){:width="740px"}

You can see the aggregated cpu metrics graph of all 5 Web UI pods by going to the OpenShift Web Console and clicking on 
**Monitoring** and then the arrow (**>**) on the left side of **web-n** under **Deployments**.

![Web UI Aggregated CPU Metrics]({% image_path fault-autoscale-metrics.png %}){:width="740px"}

When the load on Web UI disappears, after a while OpenShift scales the Web UI pods down to the minimum 
or whatever this needed to cope with the load at that point.

#### Self-healing Failed Application Pods

We looked at how to build more resilience into the applications through scaling in the 
previous sections. In this section, you will learn how to recover application pods when 
failures happen. In fact, you don't need to do anything because OpenShift automatically 
recovers failed pods when pods are not feeling healthy. The healthiness of application pods is determined via the 
[health probes]({{OPENSHIFT_DOCS_BASE}}/dev_guide/application_health.html#container-health-checks-using-probes) 
which was discussed in the previous labs.

There are three auto-healing scenarios that OpenShift handles automatically:

* Application Pod Temporary Failure: when an application pod fails and does not pass its 
[liveness health probe]({{OPENSHIFT_DOCS_BASE}}/dev_guide/application_health.html#container-health-checks-using-probes),  
OpenShift restarts the pod in order to give the application a chance to recover and start functioning 
again. Issues such as deadlocks, memory leaks, network disturbance and more are all examples of issues 
that can most likely be resolved by restarting the application despite the potential bug remaining in the 
application.

* Application Pod Permanent Failure: when an application pod fails and does not pass its 
[readiness health probe]({{OPENSHIFT_DOCS_BASE}}/dev_guide/application_health.html#container-health-checks-using-probes), 
it signals that the failure is more severe and restart is unlikely to help to mitigate the issue. OpenShift then 
removes the application pod from the load-balancer to prevent sending traffic to it.

* Application Pod Removal: if an instance of the application pods gets removed, OpenShift automatically 
starts new identical application pods based on the same container image and configuration so that the 
specified number of instances are running at all times. An example of a removed pod is when an entire 
node (virtual or physical machine) crashes and is removed from the cluster.

> OpenShift is quite orderly in this regard and if extra instances of the application pod would start running, 
> it would kill the extra pods so that the number of running instances matches what is configured on the deployment 
> config.

All of the above comes out-of-the-box and don't need any extra configuration. Remove the Catalog 
pod to verify how OpenShift starts the pod again. First, check the Catalog pod that is running:

~~~shell
$ oc get pods -l deploymentconfig=catalog

NAME              READY     STATUS    RESTARTS   AGE
catalog-3-xf111   1/1       Running   0          42m
~~~

The `-l` options tells the command to list pods that have the `deploymentconfig=catalog` label 
assigned to them. You can see pods labels using `oc get pods --show-labels` command.

Delete the Catalog pod. 

~~~shell
oc delete pods -l deploymentconfig=catalog
~~~

You need to be fast for this one! List the Catalog pods again immediately:

~~~shell
$ oc get pods -l deploymentconfig=catalog

NAME              READY     STATUS              RESTARTS   AGE
catalog-3-5dx5d   0/1       ContainerCreating   0          1s
catalog-3-xf111   0/1       Terminating         0          4m
~~~

As the Catalog pod is being deleted, OpenShift notices the lack of 1 pod and starts a new Catalog 
pod automatically.

#### Preventing Cascading Failures with Circuit Breakers

In this lab so far you have been looking at how to make sure the application pod is running, can scale to accommodate 
user load and recovers from failures. However failures also happen in the downstream services that an application 
is dependent on. It's not uncommon that the whole application fails or slows down because one of the downstream 
services consumed by the application is not responsive or responds slowly.

[Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html) is a pattern to address this issue and while 
it became popular with microservice architecture, it's a useful pattern for all applications that depend on other 
services.

The idea behind the circuit breaker is that you wrap the API calls to downstream services in a circuit breaker 
object, which monitors for failures. Once the service invocation fails a certain number of times, the circuit 
breaker flips open, and all further calls to the circuit breaker return with an error or a fallback logic 
without making the call to the unresponsive API. After a certain period, the circuit breaker will allow a call 
to the downstream service to test the waters. If the call is successful, the circuit breaker closes and would call 
the downstream service on consequent calls.

![Circuit Breaker]({% image_path fault-circuit-breaker.png %}){:width="300px"}

Spring Boot and WildFly Swarm provide convenient integration with [Hystrix](https://github.com/Netflix/Hystrix) 
which is a framework that provides circuit breaker functionality. Eclipse Vert.x, in addition to integration 
with Hystrix, provides built-in support for circuit breakers.

Let's take the Inventory service down and see what happens to the CoolStore online shop.

~~~shell
$ oc scale dc/inventory --replicas=0
~~~

Now point your browser at the Web UI route url.

> You can find the Web UI route url in the OpenShift Web Console above the `web` pod or 
> using the `oc get routes` command.

![CoolStore Without Circuit Breaker]({% image_path fault-coolstore-no-cb.png %}){:width="840px"}

Although only the Inventory service is down, there are no products displayed in the online store because 
the Inventory service call failure propagates and causes the entire API Gateway to blow up! 

The CoolStore online shop cannot function without the products list, however the inventory status is not a 
crucial bit in the shopping experience. Let's add a circuit breaker for calls to the Inventory service and 
provide a default inventory status when the Inventory service is not responsive.

In the `gateway-vertx` project, open `src/main/java/com/redhat/cloudnative/gateway/GatewayVerticle.java` and 
replace its code it with the following code:

~~~java
package com.redhat.cloudnative.gateway;

import io.vertx.circuitbreaker.CircuitBreakerOptions;
import io.vertx.core.http.HttpMethod;
import io.vertx.core.json.Json;
import io.vertx.core.json.JsonObject;
import io.vertx.ext.web.client.WebClientOptions;
import io.vertx.rxjava.circuitbreaker.CircuitBreaker;
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
    private CircuitBreaker circuit;

    @Override
    public void start() {

        circuit = CircuitBreaker.create("inventory-circuit-breaker", vertx,
            new CircuitBreakerOptions()
                .setFallbackOnFailure(true)
                .setMaxFailures(3)
                .setResetTimeout(5000)
                .setTimeout(1000)
        );

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
                        circuit.rxExecuteCommandWithFallback(
                            future ->
                                inventory.get("/api/inventory/" + product.getString("itemId")).as(BodyCodec.jsonObject())
                                    .rxSend()
                                    .map(resp -> {
                                        if (resp.statusCode() != 200) {
                                            LOG.warn("Inventory error for {}: status code {}",
                                                    product.getString("itemId"), resp.statusCode());
                                        }
                                        return product.copy().put("availability", 
                                            new JsonObject().put("quantity", resp.body().getInteger("quantity")));
                                    })
                                    .subscribe(
                                        future::complete,
                                        future::fail),
                            error -> {
                                LOG.error("Inventory error for {}: {}", product.getString("itemId"), error.getMessage());
                                return product;
                            }
                        ))
                    .toList().toSingle()
            )
            .subscribe(
                list -> rc.response().end(Json.encodePrettily(list)),
                error -> rc.response().end(new JsonObject().put("error", error.getMessage()).toString())
            );
    }
}
~~~

The above code is quite similar to the previous code however it wraps the calls to the Inventory 
service in a `CircuitBreaker` using the built-in circuit breaker in Vert.x. The circuit breaker 
is configured to flip open after 3 failures and time out on the 
calls after 1 second. 

The `circuit.rxExecuteCommandWithFallback(...)` method, defines the fallback logic for 
when the circuit is open and logs an error without calling the Inventory service in those 
scenarios.

Build and package the Gateway service using Maven by clicking on **BUILD > build** from the commands palette.

![Maven Build]({% image_path eclipse-che-commands-build.png %}){:width="340px"}

Although you can use the **DEPLOY > fabric8:deploy** from the commands palette, you 
can also trigger a new container image build on OpenShift using 
the `oc start-build` command which allows you to build container images directly from the application 
archives (`jar`, `war`, etc) without the need to have access to the source code for example by downloading 
the `jar` file form the Maven repository (e.g. Nexus or Artifactory).

~~~shell
$ oc start-build gateway-s2i --from-file=labs/gateway-vertx/target/gateway-1.0-SNAPSHOT.jar
~~~

As soon as the new `gateway` container image is built, OpenShift deploys the new image automatically 
thanks to the [deployment triggers]({{OPENSHIFT_DOCS_BASE}}/dev_guide/deployments/basic_deployment_operations.html#triggers) 
defined on the `gateway` deployment config.

Let's try the Web UI again in the browser while the Inventory service is still down.

![CoolStore With Circuit Breaker]({% image_path fault-coolstore-with-cb.png %}){:width="840px"}

It looks better now! The Inventory service failure is contained and the inventory status is removed from the 
user interface and allows the CoolStore online shop to continue functioning and accept orders. Selling an 
out-of-stock product to a few customers can simply be resolved by a discount coupons while 
losing the trust of all visiting customers due to a crashed online store is not so easily repairable!

Scale the Inventory service back up before moving on to the next labs.

~~~shell
$ oc scale dc/inventory --replicas=1
~~~

Well done! Let's move on to the next.

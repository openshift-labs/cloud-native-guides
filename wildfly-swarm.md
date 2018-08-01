## Enterprise Microservices with WildFly Swarm

In this lab you will learn about building microservices using WildFly Swarm.

#### What is WildFly Swarm?

Java EE applications are traditionally created as an `ear` or `war` archive including all 
dependencies and deployed in an application server. Multiple Java EE applications can and 
were typically deployed in the same application server. This model is well understood in 
the development teams and has been used over the past several years.

WildFly Swarm offers an innovative approach to packaging and running Java EE applications by 
packaging them with just enough of the Java EE server runtime to be able to run them directly 
on the JVM using `java -jar`. For more details on various approaches to packaging Java 
applications, read [this blog post](https://developers.redhat.com/blog/2017/08/24/the-skinny-on-fat-thin-hollow-and-uber).

WildFly Swarm is based on WildFly and it's compatible with 
MicroProfile, which is a community effort to standardize the subset of Java EE standards 
such as JAX-RS, CDI and JSON-P that are useful for building microservices applications.

Since WildFly Swarm is based on Java EE standards, it significantly simplifies refactoring 
existing Java EE applications to microservices and allows much of the existing code-base to be 
reused in the new services.

#### WildFly Swarm Maven Project 

The `inventory-wildfly-swarm` project has the following structure which shows the components of 
the WildFly Swarm project laid out in different subdirectories according to Maven best practices:

~~~shell
├── pom.xml            # The Maven project file
└── src
    └── main
        └── java       # The source code to the project
        └── resources  # The static resource files and configurations
~~~

This is a minimal Java EE project with support for JAX-RS for building RESTful services and JPA for connecting
to a database. [JAX-RS](https://docs.oracle.com/javaee/7/tutorial/jaxrs.htm) is one of Java EE standards that uses Java annotations to simplify the development of RESTful web services. [Java Persistence API (JPA)](https://docs.oracle.com/javaee/7/tutorial/partpersist.htm) is another Java EE standard that provides Java developers with an object/relational mapping facility for managing relational data in Java applications.

This project currently contains no code other than the main class for exposing a single 
RESTful application defined in `InventoryApplication.java`. 

Examine `src/main/java/com/redhat/cloudnative/inventory/InventoryApplication.java`

~~~java
package com.redhat.cloudnative.inventory;

import javax.ws.rs.ApplicationPath;
import javax.ws.rs.core.Application;

@ApplicationPath("/")
public class InventoryApplication extends Application {
}
~~~

Run the Maven build to make sure the skeleton project builds successfully. You should get a `BUILD SUCCESS` message 
in the build logs, otherwise the build has failed.

> Make sure to run the `package` Maven goal and not `install`. The latter would 
> download a lot more dependencies and do things you don't need yet!

~~~shell
$ cd inventory-wildfly-swarm
$ mvn package

...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 10.757 s
[INFO] Finished at: 2017-07-24T10:49:54+07:00
[INFO] Final Memory: 44M/481M
[INFO] ------------------------------------------------------------------------
~~~

Once built, the resulting *jar* is located in the `target/` directory:

~~~shell
$ ls target/*.jar
target/inventory-1.0-SNAPSHOT-swarm.jar
~~~

This is an uber-jar with all the dependencies required packaged in the *jar* to enable running the 
application with `java -jar`. WildFly Swarm also creates a *war* packaging as a standard Java EE web app 
that could be deployed to any Java EE app server (for example, JBoss EAP, or its upstream WildFly project). 

Now let's write some code and create a domain model and a RESTful endpoint to create the Inventory service:

![Inventory RESTful Service]({% image_path wfswarm-inventory-arch.png %}){:width="500px"}

#### Create a Domain Model

Use your favorite text-editor (we &hearts; Visual Studio Code and Sublime) or IDE (JBoss Developer 
Studio is our favorite) to create a new Java class named `Inventory.java` in 
`com.redhat.cloudnative.inventory` package with the below code and 
following fields: `itemId` and `quantity`

~~~java
package com.redhat.cloudnative.inventory;

import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Table;
import javax.persistence.UniqueConstraint;
import java.io.Serializable;

@Entity
@Table(name = "INVENTORY", uniqueConstraints = @UniqueConstraint(columnNames = "itemId"))
public class Inventory implements Serializable {
    @Id
    private String itemId;

    private int quantity;

    public Inventory() {
    }

    public String getItemId() {
        return itemId;
    }

    public void setItemId(String itemId) {
        this.itemId = itemId;
    }

    public int getQuantity() {
        return quantity;
    }

    public void setQuantity(int quantity) {
        this.quantity = quantity;
    }

    @Override
    public String toString() {
        return "Inventory [itemId='" + itemId + '\'' + ", quantity=" + quantity + ']';
    }
}
~~~

Review the `Inventory` domain model and note the JPA annotations on this class. `@Entity` marks 
the class as a JPA entity, `@Table` customizes the table creation process by defining a table 
name and database constraint and `@Id` marks the primary key for the table.

WildFly Swarm configuration is done to a large extend through detecting the intent of the 
developer and automatically adding the required dependencies configurations to make sure it can 
get out of the way and developers can be productive with their code rather than Googling for 
configuration snippets. As an example, configuration database access with JPA is composed of 
the following steps:

1. Adding the `org.wildfly.swarm:jpa` dependency to `pom.xml` 
2. Adding the database driver (e.g. `org.postgresql:postgresql`) to `pom.xml`
3. Adding database connection details in `src/main/resources/project-stages.yml`

Examine `pom.xml` and note the `org.wildfly.swarm:jpa` that is already added to enable JPA:

~~~xml
<dependency>
    <groupId>org.wildfly.swarm</groupId>
    <artifactId>jpa</artifactId>
</dependency>
~~~

Examine `src/main/resources/META-INF/persistence.xml` to see the JPA datasource configuration 
for this project. Also note that the configurations uses `META-INF/load.sql` to import 
initial data into the database.

Examine `src/main/resources/project-stages.yml` to see the database connection details. 
An in-memory H2 database is used in this lab for local development and in the following 
labs will be replaced with a PostgreSQL database. Be patient! More on that later.

#### Create a RESTful Service

WildFly Swarm uses JAX-RS standard for building REST services. Create a new Java class named 
`InventoryResource.java` in `com.redhat.cloudnative.inventory` package with the following content:

~~~java
package com.redhat.cloudnative.inventory;

import javax.enterprise.context.ApplicationScoped;
import javax.persistence.*;
import javax.ws.rs.*;
import javax.ws.rs.core.MediaType;

@Path("/")
@ApplicationScoped
public class InventoryResource {
    @PersistenceContext(unitName = "InventoryPU")
    private EntityManager em;

    @GET
    @Path("/api/inventory/{itemId}")
    @Produces(MediaType.APPLICATION_JSON)
    public Inventory getAvailability(@PathParam("itemId") String itemId) {
        Inventory inventory = em.find(Inventory.class, itemId);
        return inventory;
    }
}
~~~

The above REST services defines an endpoint that is accessible via `HTTP GET` at 
for example `/api/inventory/329299` with 
the last path param being the product id which we want to check its inventory status.

Build and package the Inventory service using Maven

~~~shell
$ mvn package
~~~

Using WildFly Swarm maven plugin, you can conveniently run the application locally and test the endpoint.

~~~shell
$ mvn wildfly-swarm:run
~~~

> Alternatively, you can run the application using the uber-jar produced during the
> Maven build: `java -jar target/inventory-1.0-SNAPSHOT-swarm.jar`

Once you see `WildFly Swarm is Ready` in the logs, the Inventory service is up and running and you can access the 
inventory REST API. Let’s test it out using `curl` in a new terminal window:

~~~shell
$ curl http://localhost:9001/api/inventory/329299

{"itemId":"329299","quantity":35}
~~~

The REST API returned a JSON object representing the inventory count for this product. Congratulations!

Stop the service by pressing `CTRL-C` in the terminal window.

#### Deploy WildFly Swarm on OpenShift

It’s time to build and deploy our service on OpenShift.

OpenShift [Source-to-Image (S2I)]({{OPENSHIFT_DOCS_BASE}}/architecture/core_concepts/builds_and_image_streams.html#source-build) 
feature can be used to build a container image from your project. OpenShift 
S2I uses the [supported OpenJDK container image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift) to build the final container image of the 
Inventory service by uploading the WildFly Swam uber-jar from the `target` folder to 
the OpenShift platform. 

Maven projects can use the [Fabric8 Maven Plugin](https://maven.fabric8.io) in order 
to use OpenShift S2I for building 
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

* The Inventory uber-jar is built using WildFly Swarm
* A container image is built on OpenShift containing the Inventory uber-jar and JDK
* All necessary objects are created within the OpenShift project to deploy the Inventory service

Once this completes, your project should be up and running. OpenShift runs the different components of 
the project in one or more pods which are the unit of runtime deployment and consists of the running 
containers for the project. 

Let's take a moment and review the OpenShift resources that are created for the Inventory REST API:

* **Build Config**: `inventory-s2i` build config is the configuration for building the Inventory 
container image from the inventory source code or JAR archive
* **Image Stream**: `inventory` image stream is the virtual view of all inventory container 
images built and pushed to the OpenShift integrated registry.
* **Deployment Config**: `inventory` deployment config deploys and redeploys the Inventory container 
image whenever a new Inventory container image becomes available
* **Service**: `inventory` service is an internal load balancer which identifies a set of 
pods (containers) in order to proxy the connections it receives to them. Backing pods can be 
added to or removed from a service arbitrarily while the service remains consistently available, 
enabling anything that depends on the service to refer to it at a consistent address (service name 
or IP).
* **Route**: `inventory` route registers the service on the built-in external load-balancer 
and assigns a public DNS name to it so that it can be reached from outside OpenShift cluster.

You can review the above resources in the OpenShift Web Console or using `oc describe` command:

> `bc` is the short-form of `buildconfig` and can be interchangeably used 
> instead of it with the OpenShift CLI. The same goes for `is` instead 
> of `imagestream`, `dc` instead of `deploymentconfig` and `svc` instead of `service`.

~~~shell
$ oc describe bc inventory-s2i
$ oc describe is inventory
$ oc describe dc inventory
$ oc describe svc inventory
$ oc describe route inventory
~~~

You can see the exposed DNS url for the Inventory service in the OpenShift Web Console or using 
OpenShift CLI:

~~~shell
$ oc get routes

NAME        HOST/PORT                                        PATH       SERVICES  PORT  TERMINATION   
inventory   inventory-{{COOLSTORE_PROJECT}}.roadshow.openshiftapps.com   inventory  8080            None
~~~

Copy the route url for the Inventory service and verify the API Gateway service works using `curl`:

> The route urls in your project would be different from the ones in this lab guide! Use the one from yor project.

~~~shell
$ curl http://{{INVENTORY_ROUTE_HOST}}/api/inventory/329299

{"itemId":"329299","quantity":35}
~~~

Well done! You are ready to move on to the next lab.
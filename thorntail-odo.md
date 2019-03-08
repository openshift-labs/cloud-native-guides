## Enterprise Microservices with Thorntail

In this lab you will learn about building microservices using Thorntail.

![CoolStore Architecture]({% image_path coolstore-arch-inventory.png %}){:width="200px"}

#### What is Thorntail?

Java EE applications are traditionally created as an `ear` or `war` archive including all 
dependencies and deployed in an application server. Multiple Java EE applications can and 
were typically deployed in the same application server. This model is well understood in 
development teams and has been used over the past several years.

Thorntail offers an innovative approach to packaging and running Java EE applications by 
packaging them with just enough of the Java EE server runtime to be able to run them directly 
on the JVM using `java -jar`. For more details on various approaches to packaging Java 
applications, read [this blog post](https://developers.redhat.com/blog/2017/08/24/the-skinny-on-fat-thin-hollow-and-uber).

Thorntail is based on WildFly and it's compatible with 
MicroProfile, which is a community effort to standardize the subset of Java EE standards 
such as JAX-RS, CDI and JSON-P that are useful for building microservices applications.

Since Thorntail is based on Java EE standards, it significantly simplifies refactoring 
existing Java EE applications to microservices and allows much of the existing code-base to be 
reused in the new services.

#### Thorntail Maven Project 

The `inventory-thorntail` project has the following structure which shows the components of 
the Thorntail project laid out in different subdirectories according to Maven best practices:

![Inventory Project]({% image_path thorntail-inventory-project.png %}){:width="200px"}

This is a minimal Java EE project with support for JAX-RS for building RESTful services and JPA for connecting
to a database. [JAX-RS](https://docs.oracle.com/javaee/7/tutorial/jaxrs.htm) is one of Java EE standards that uses Java annotations to simplify the development of RESTful web services. [Java Persistence API (JPA)](https://docs.oracle.com/javaee/7/tutorial/partpersist.htm) is another Java EE standard that provides Java developers with an object/relational mapping facility for managing relational data in Java applications.

This project currently contains no code other than the main class for exposing a single 
RESTful application defined in `InventoryApplication`. 

Examine `com.redhat.cloudnative.inventory.InventoryApplication` in the `src/main` directory:

~~~java
package com.redhat.cloudnative.inventory;

import javax.ws.rs.ApplicationPath;
import javax.ws.rs.core.Application;

@ApplicationPath("/api")
public class InventoryApplication extends Application {
}
~~~

#### Creating an Openshift Application

An application is an umbrella of components that work together to implement the overall application. OpenShift helps organize these modular applications with a concept called, appropriately enough, the application. An OpenShift application represents all of an app's components in a logical management unit.

First, create an application called `inventory` to work with:

~~~shell
$ odo app create inventory
Creating application: inventory in project: {{COOLSTORE_PROJECT}}
Switched to application: inventory in project: {{COOLSTORE_PROJECT}}
~~~

You can verify that the new application is created with the following commands:

~~~shell
$ odo app list
The project '{{COOLSTORE_PROJECT}}' has the following applications:
ACTIVE     NAME
*          inventory
~~~

#### Creating a Service Component from Binary

Run the Maven build to make sure the skeleton project builds successfully. You should get a `BUILD SUCCESS` message 
in the build logs, otherwise the build has failed.

In CodeReady Workspaces, click on **inventory-thorntail** project in the project explorer, 
and then click on Commands Palette and click on **BUILD > build**

![Maven Build]({% image_path  codeready-command-build.png %}){:width="200px"}

Once built successfully, the resulting *jar* is located in the `target/` directory:

~~~shell
$ ls labs/inventory-thorntail/target/*-thorntail.jar
labs/inventory-thorntail/target/inventory-1.0-SNAPSHOT-thorntail.jar
~~~

This is an uber-jar with all the dependencies required packaged in the *jar* to enable running the 
application with `java -jar`. Thorntail also creates a *war* packaging as a standard Java EE web app 
that could be deployed to any Java EE app server (for example, JBoss EAP, or its upstream WildFly project). 

Now, add a component named `service` of type `redhat-openjdk18-openshift:1.4` to the application `inventory` and deploy the uber-jar `inventory-1.0-SNAPSHOT-thorntail.jar`:

~~~shell
$ odo create redhat-openjdk18-openshift:1.4 service --app inventory \
--binary labs/inventory-thorntail/target/inventory-1.0-SNAPSHOT-thorntail.jar
 ✓   Checking component
 ✓   Checking component version
 ✓   Creating component service
 OK  Component 'service' was created and ports 8778/TCP,8080/TCP,8443/TCP were opened
 OK  Component 'service' is now set as active component
To push source code to the component run 'odo push'
~~~

![inventory Service Component]({% image_path thorntail-inventory-component.png %}){:width="500px"}

#### Pushing your source code

Now that the component is running, push our initial source code:

~~~shell
$ odo push service --app inventory
Pushing changes to component: service
 ✓   Waiting for pod to start
 ✓   Copying files to pod
 ✓   Building component
 OK  Changes successfully pushed to component: service
~~~

The jar file has now been pushed to the container, and the process in that container restarted.

#### Creating an OpenShift route

To access to the service outside the cluster, create an external URL (an Openshift Route) for the `inventory` application:

~~~shell
$ odo url create --app inventory --component service --port 8080
Adding URL to component: service
 OK  URL created for component: service

service - http://service-inventory-{{COOLSTORE_PROJECT}}.apps.openshiftworkshop.com
~~~

> The route urls in your project would be different from the ones in this lab guide! Use the ones from your project.

Copy the route url for the inventory service into your browser:

![inventory Service Root]({% image_path thorntail-inventory-service-root.png %}){:width="500s"}

#### Updating Component on Change

Watch for Changes and updating Component on Change. Let's run `odo watch` in a new terminal window.

~~~shell
$ odo watch service --app inventory
Waiting for something to change in /projects/labs/inventory-thorntail/target/inventory-1.0-SNAPSHOT-thorntail.jar
~~~

Now let's write some code and create a domain model and a RESTful endpoint to create the Inventory service:

![Inventory RESTful Service]({% image_path wfswarm-inventory-arch.png %}){:width="500px"}

#### Creating a Domain Model

Create a new Java class named `Inventory` in `com.redhat.cloudnative.inventory` package with the below code and 
following fields: `itemId` and `quantity`

In the project explorer in CodeReady Workspaces, right-click on **inventory-thorntail > src > main > java > com.redhat.cloudnative.inventory** and then on **New > Java Class**. Enter `Inventory` as the Java class name.

![CodeReady Workspaces - Create Java Class]({% image_path wfswarm-inventory-che-new-class.png %}){:width="700px"}

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

You don't need to press a save button! CodeReady Workspaces automatically saves the changes made to the files.

Review the `Inventory` domain model and note the JPA annotations on this class. `@Entity` marks 
the class as a JPA entity, `@Table` customizes the table creation process by defining a table 
name and database constraint and `@Id` marks the primary key for the table.

Thorntail configuration is done to a large extent through detecting the intent of the 
developer and automatically adding the required dependencies configurations to make sure it can 
get out of the way and developers can be productive with their code rather than Googling for 
configuration snippets. As an example, configuration database access with JPA is composed of 
the following steps:

1. Adding the `io.thorntail:jpa` dependency to `pom.xml` 
2. Adding the database driver (e.g. `org.postgresql:postgresql`) to `pom.xml`
3. Adding database connection details in `src/main/resources/project-default.yml`

Edit the `pom.xml` file and add the `io.thorntail:jpa` dependency to enable JPA:

~~~xml
<dependency>
    <groupId>io.thorntail</groupId>
    <artifactId>jpa</artifactId>
</dependency>
~~~

Examine `src/main/resources/META-INF/persistence.xml` to see the JPA datasource configuration 
for this project. Also note that the configurations uses `META-INF/load.sql` to import 
initial data into the database.

Examine `src/main/resources/project-default.yml` to see the database connection details. 
An in-memory H2 database is used in this lab for local development and in the following 
labs will be replaced with a PostgreSQL database. Be patient! More on that later.

#### Creating a RESTful Service

Thorntail uses JAX-RS standard for building REST services. In the project explorer in CodeReady Workspaces, replace the Java class named  `InventoryResource` from **inventory-thorntail > src > main > java > com.redhat.cloudnative.inventory** with the following content:

~~~java
package com.redhat.cloudnative.inventory;

import javax.enterprise.context.ApplicationScoped;
import javax.persistence.*;
import javax.ws.rs.*;
import javax.ws.rs.core.MediaType;

@Path("/inventory")
@ApplicationScoped
public class InventoryResource {
    @PersistenceContext(unitName = "InventoryPU")
    private EntityManager em;

    @GET
    @Path("/{itemId}")
    @Produces(MediaType.APPLICATION_JSON)
    public Inventory getAvailability(@PathParam("itemId") String itemId) {
        Inventory inventory = em.find(Inventory.class, itemId);
        return inventory;
    }
}
~~~

The above REST service defines an endpoint that is accessible via `HTTP GET` at 
for example `/api/inventory/329299` with 
the last path param being the product id which we want to check its inventory status.

Build and package the Inventory service by clicking on the commands palette and then **BUILD > build**

![Maven Build]({% image_path  codeready-command-build.png %}){:width="200px"}

> Make sure **inventory-thorntail** project is highlighted in the project explorer

Once successfully built, your new version of the jar will be pushed automatically into the Inventory Component thanks to the `odo watch` command. You should see following logs in the Terminal where you ran the `odo watch` command.

~~~shell
File /projects/labs/inventory-thorntail/target/inventory-1.0-SNAPSHOT-thorntail.jar changed
Pushing files...
 ✓   Waiting for pod to start
 ✓   Copying files to pod
 ✓   Building component
Waiting for something to change in /projects/labs/inventory-thorntail/target/inventory-1.0-SNAPSHOT-thorntail.jar
~~~

Now, you can access the Inventory REST API. Let’s test it out using `curl` in a new terminal window:

~~~shell
$ odo url list --component service --app inventory
Found the following URLs for component service in application inventory:
NAME                  URL                                                                   PORT
service     http://service-inventory-{{COOLSTORE_PROJECT}}.apps.openshiftworkshop.com     8080

$ curl http://service-inventory-{{COOLSTORE_PROJECT}}apps.openshiftworkshop.com/api/inventory/329299
{"itemId":"329299","quantity":35}
~~~

Well done! You are ready to move on to the next lab.

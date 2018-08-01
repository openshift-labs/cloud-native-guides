## Microservices with Spring Boot

In this lab you will learn about Spring Boot and how you can build microservices 
using Spring Boot and JBoss technologies. During this lab, you will create a REST API for 
the Catalog service in order to provide a list of products for the CoolStore online shop.

#### What is Spring Boot?

Spring Boot is an opinionated framework that makes it easy to create stand-alone Spring based 
applications with an embedded web containers such as Tomcat (or JBoss Web Server), Jetty and Undertow 
that you can run directly on the JVM using `java -jar`. Spring Boot also allows producing a war 
file that can be deployed on stand-alone web containers.

The opinionated approach means many choices about Spring platform and third-party libraries 
are already made by Spring Boot so that you can get started with minimum effort and configuration.

#### Spring Boot Maven Project 

The `catalog-spring-boot` project has the following structure which shows the components of 
the Spring Boot project laid out in different subdirectories according to Maven best practices:


Once loaded, you should see the following files and be able to navigate amongst the files. The 
components of the Spring Boot project are laid out in different subdirectories according to Maven best practices:

![Catalog Project]({% image_path springboot-catalog-project.png %}){:width="340px"}

This is a minimal Spring Boot project with support for RESTful services and Spring Data with JPA for connecting
to a database. This project currently contains no code other than the main class, `CatalogApplication`
which is there to bootstrap the Spring Boot application.

Examine `com/redhat/cloudnative/catalog/CatalogApplication` in the `src/main` directory:

~~~java
package com.redhat.cloudnative.catalog;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class CatalogApplication {

    public static void main(String[] args) {
        SpringApplication.run(CatalogApplication.class, args);
    }
}
~~~

The database is configured using the Spring application configuration file which is located at 
`src/main/resources/application.properties`. Examine this file to see the database connection details 
and note that an in-memory H2 database is used in this lab for local development and will be replaced
with a PostgreSQL database in the following labs. Be patient! More on that later.

You can use Maven to make sure the skeleton project builds successfully. You should get a `BUILD SUCCESS` message 
in the build logs, otherwise the build has failed.

In Eclipse Che, click on **catalog-spring-boot** project in the project explorer, 
and then click on Commands Palette and click on **BUILD > build**

![Maven Build]({% image_path eclipse-che-commands-build.png %}){:width="340px"}

Once successfully built, the resulting `jar` is located in the `target/` directory:

~~~shell
$ ls labs/catalog-spring-boot/target/*.jar

labs/catalog-spring-boot/target/catalog-1.0-SNAPSHOT.jar
~~~

This is an uber-jar with all the dependencies required packaged in the `jar` to enable running the 
application with `java -jar`.

Now that the project is ready, let's get coding and create a domain model, data repository, and a  
RESTful endpoint to create the Catalog service:

![Catalog RESTful Service]({% image_path springboot-catalog-arch.png %}){:width="640px"}

#### Create the Domain Model

Create a new Java class named `Product` in the `com.redhat.cloudnative.catalog` package with the below code and 
following fields: `itemId`, `name`, `desc` and `price`

In the project explorer in Eclipse Che, right-click on **catalog-spring-boot > src > main > java > com.redhat.cloudnative.catalog** and then on **New > Java Class**. Enter `Product` as the Java class name.


~~~java
package com.redhat.cloudnative.catalog;

import java.io.Serializable;

import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Table;
import javax.persistence.UniqueConstraint;

@Entity
@Table(name = "PRODUCT", uniqueConstraints = @UniqueConstraint(columnNames = "itemId"))
public class Product implements Serializable {
  
  @Id
  private String itemId;
  
  private String name;
  
  private String description;
  
  private double price;

  public Product() {
  }
  
  public String getItemId() {
    return itemId;
  }

  public void setItemId(String itemId) {
    this.itemId = itemId;
  }

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }

  public String getDescription() {
    return description;
  }

  public void setDescription(String description) {
    this.description = description;
  }

  public double getPrice() {
    return price;
  }

  public void setPrice(double price) {
    this.price = price;
  }

  @Override
  public String toString() {
    return "Product [itemId=" + itemId + ", name=" + name + ", price=" + price + "]";
  }
}
~~~

Review the `Product` domain model and note the JPA annotations on this class. `@Entity` marks the 
class as a JPA entity, `@Table` customizes the table creation process by defining a table 
name and database constraint and `@Id` marks the primary key for the table

#### Create a Data Repository

Spring Data repository abstraction simplifies dealing with data models in Spring applications by 
reducing the amount of boilerplate code required to implement data access layers for various 
persistence stores. [Repository and its sub-interfaces](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.core-concepts) 
are the central concept in Spring Data which is a marker interface to provide 
data manipulation functionality for the entity class that is being managed. When the application starts, 
Spring finds all interfaces marked as repositories and for each interface found, the infrastructure 
configures the required persistent technologies and provides an implementation for the repository interface.

Create a new Java interface named `ProductRepository` in `com.redhat.cloudnative.catalog` package 
and extend [CrudRepository](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html) interface in order to indicate to Spring that you want to expose a 
complete set of methods to manipulate the entity.

In the project explorer in Eclipse Che, right-click on **catalog-spring-boot > src > main > java > com.redhat.cloudnative.catalog** and then on **New > Java Class** and paste the following code:


~~~java
package com.redhat.cloudnative.catalog;

import org.springframework.data.repository.CrudRepository;

public interface ProductRepository extends CrudRepository<Product, String> {
}
~~~

That's it! Now that you have a domain model and a repository to retrieve the domain mode, let's create a 
RESTful service that returns the list of products.

#### Create a RESTful Service

Spring Boot uses Spring Web MVC as the default RESTful stack in Spring applications. Create 
a new Java class named `CatalogController` in `com.redhat.cloudnative.catalog` package with 
the following content by right-clicking on **catalog-spring-boot > src > main > java > com.redhat.cloudnative.catalog** and 
then clicking on **New > Java Class**:

~~~java
package com.redhat.cloudnative.catalog;

import java.util.*;
import java.util.stream.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.*;

@Controller
@RequestMapping(value = "/api/catalog")
public class CatalogController {
    @Autowired
    private ProductRepository repository;

    @ResponseBody
    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    public List<Product> getAll() {
        Spliterator<Product> products = repository.findAll().spliterator();
        return StreamSupport.stream(products, false).collect(Collectors.toList());
    }
}
~~~

The above REST services defines an endpoint that is accessible via `HTTP GET` at `/api/catalog`. Notice 
the `repository` field on the controller class which is used to retrieve the list of products. Spring Boot 
automatically provides an implementation for `ProductRepository` at runtime and 
[injects it into the controller using the `@Autowire` annotation](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-spring-beans-and-dependency-injection.html).

Build and package the Catalog service using Maven by clicking on **BUILD > build** from the commands palette.

![Maven Build]({% image_path eclipse-che-commands-build.png %}){:width="340px"}

Using Spring Boot maven plugin, you can conveniently run the application in Eclipse Che and 
test the endpoint. In Eclipse Che, click on the run icon and then on **run spring-boot**. 

> You can also run the inventory service in Eclipse Che using the commands palette and then **run > run spring-boot**
 
When you see `Started CatalogApplication` in the logs, you can access the 
Catalog REST API. Let’s test it out using `curl` in a new terminal window:

~~~shell
$ curl http://localhost:9000/api/catalog

[{"itemId":"329299","name":"Red Fedora","desc":"Official Red Hat Fedora","price":34.99},...]
~~~

You can also use the preview url that Eclipse Che has generated for you to be able to test service 
directly in the browser. Append the path `/api/catalog` at the end of the preview url and try 
it in your browser in a new tab.

![Preview URL]({% image_path springboot-che-preview-browser.png %}){:width="900px"}


The REST API returned a JSON object representing the product list. Congratulations!

Stop the Inventory service by clicking on the stop icon near **run spring-boot** in the **Machines** window.

#### Deploy Spring Boot on OpenShift

It’s time to build and deploy our service on OpenShift. 

OpenShift [Source-to-Image (S2I)]({{OPENSHIFT_DOCS_BASE}}/architecture/core_concepts/builds_and_image_streams.html#source-build) 
feature can be used to build a container image from your project. OpenShift 
S2I uses the [supported OpenJDK container image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift) to build the final container image 
of the Catalog service by uploading the Spring Boot uber-jar from the `target` 
folder to the OpenShift platform. 

Maven projects can use the [Fabric8 Maven Plugin](https://maven.fabric8.io) in order to use OpenShift S2I for building 
the container image of the application from within the project. This maven plugin is a Kubernetes/OpenShift client 
able to communicate with the OpenShift platform using the REST endpoints in order to issue the commands 
allowing to build a project, deploy it and finally launch a docker process as a pod.

To build and deploy the Catalog service on OpenShift using the `fabric8` maven plugin, 
which is already configured in Eclipse Che, from the commands palette, click on **DEPLOY > fabric8:deploy**

![Fabric8 Deploy]({% image_path eclipse-che-commands-deploy.png %}){:width="340px"}

During the deployment, you might see that Fabric8 Maven Plugin throws an `java.util.concurrent.RejectedExecutionException` 
exception. This is due to [a bug](https://github.com/fabric8io/kubernetes-client/issues/1035) in one of Fabric8 Maven Plugin 
dependencies which is being worked on right now and will be fixed soon. You can ignore this exception for now. The deployment 
nevertheless succeeds.

`fabric8:deploy` will cause the following to happen:

* The Catalog uber-jar is built using Spring Boot
* A container image is built on OpenShift containing the Catalog uber-jar and JDK
* All necessary objects are created within the OpenShift project to deploy the Catalog service

Once this completes, your project should be up and running. OpenShift runs the different components of 
the project in one or more pods which are the unit of runtime deployment and consists of the running 
containers for the project. 

Let's take a moment and review the OpenShift resources that are created for the Catalog REST API:

* **Build Config**: `catalog-s2i` build config is the configuration for building the Catalog 
container image from the catalog source code or JAR archive
* **Image Stream**: `catalog` image stream is the virtual view of all catalog container 
images built and pushed to the OpenShift integrated registry.
* **Deployment Config**: `catalog` deployment config deploys and redeploys the Catalog container 
image whenever a new Catalog container image becomes available
* **Service**: `catalog` service is an internal load balancer which identifies a set of 
pods (containers) in order to proxy the connections it receives to them. Backing pods can be 
added to or removed from a service arbitrarily while the service remains consistently available, 
enabling anything that depends on the service to refer to it at a consistent address (service name 
or IP).
* **Route**: `catalog` route registers the service on the built-in external load-balancer 
and assigns a public DNS name to it so that it can be reached from outside OpenShift cluster.

You can review the above resources in the OpenShift Web Console or using `oc describe` command:

> `bc` is the short-form of `buildconfig` and can be interchangeably used instead of it with the 
> OpenShift CLI. The same goes for `is` instead of `imagestream`, `dc` instead of`deploymentconfig` 
> and `svc` instead of `service`.

~~~shell
$ oc describe bc catalog-s2i
$ oc describe is catalog
$ oc describe dc catalog
$ oc describe svc catalog
$ oc describe route catalog
~~~

You can see the expose DNS url for the Catalog service in the OpenShift Web Console or using 
OpenShift CLI:

~~~shell
$ oc get routes

NAME        HOST/PORT                                        PATH       SERVICES  PORT  TERMINATION   
catalog     catalog-{{COOLSTORE_PROJECT}}.roadshow.openshiftapps.com     catalog    8080            None
inventory   inventory-{{COOLSTORE_PROJECT}}.roadshow.openshiftapps.com   inventory  8080            None
~~~

Copy the route url for the Catalog service and verify the Catalog service works using `curl`:

> The route urls in your project would be different from the ones in this lab guide! Use the ones from yor project.

~~~shell
$ curl http://{{CATALOG_ROUTE_HOST}}/api/catalog

[{"itemId":"329299","name":"Red Fedora","desc":"Official Red Hat Fedora","price":34.99},...]
~~~

Well done! You are ready to move on to the next lab.

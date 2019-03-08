## Microservices with Spring Boot

In this lab you will learn about Spring Boot and how you can build microservices using Spring Boot and JBoss technologies. During this lab, you will create a REST API for the Catalog service in order to provide a list of products for the CoolStore online shop.

![CoolStore Architecture]({% image_path coolstore-arch-catalog.png %}){:width="500px"}

#### What is Spring Boot?

Spring Boot is an opinionated framework that makes it easy to create stand-alone Spring based applications with an embedded web containers such as Tomcat (or JBoss Web Server), Jetty and Undertow that you can run directly on the JVM using `java -jar`. Spring Boot also allows producing a war file that can be deployed on stand-alone web containers.

The opinionated approach means many choices about Spring platform and third-party libraries are already made by Spring Boot so that you can get started with minimum effort and configuration.

#### Spring Boot Maven Project 

The `catalog-spring-boot` project has the following structure which shows the components of the Spring Boot project laid out in different subdirectories according to Maven best practices:


Once loaded, you should see the following files and be able to navigate amongst the files. The components of the Spring Boot project are laid out in different subdirectories according to Maven best practices:

![Catalog Project]({% image_path springboot-catalog-project.png %}){:width="200px"}

This is a minimal Spring Boot project with support for RESTful services and Spring Data with JPA for connecting to a database. This project currently contains no code other than the main class, `CatalogApplication` which is there to bootstrap the Spring Boot application.

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

The database is configured using the Spring application configuration file which is located at `src/main/resources/application.properties`. Examine this file to see the database connection details and note that an in-memory H2 database is used in this lab for local development and will be replaced with a PostgreSQL database in the following labs. Be patient! More on that later.

#### Creating an Openshift Application

An application is an umbrella of components that work together to implement the overall application. OpenShift helps organize these modular applications with a concept called, appropriately enough, the application. An OpenShift application represents all of an app's components in a logical management unit.

First, create an application called `catalog` to work with:

~~~shell
$ odo app create catalog
Creating application: catalog in project: {{COOLSTORE_PROJECT}}
Switched to application: catalog in project: {{COOLSTORE_PROJECT}}
~~~

You can verify that the new application is created with the following commands:

~~~shell
$ odo app list
The project '{{COOLSTORE_PROJECT}}' has the following applications:
ACTIVE     NAME
           inventory
*          catalog
~~~

#### Creating a Service Component from Binary

Run Maven to make sure the skeleton project builds successfully. You should get a `BUILD SUCCESS` message in the build logs, otherwise the build has failed.

In CodeReady Workspaces, click on **catalog-spring-boot** project in the project explorer, and then click on Commands Palette and click on **BUILD > build**

![Maven Build]({% image_path  codeready-command-build.png %}){:width="200px"}

Once successfully built, the resulting `jar` is located in the `target/` directory:

~~~shell
$ ls labs/catalog-spring-boot/target/*.jar

labs/catalog-spring-boot/target/catalog-1.0-SNAPSHOT.jar
~~~

This is an uber-jar with all the dependencies required packaged in the `jar` to enable running the application with `java -jar`.

Now, add a component named `service` of type `redhat-openjdk18-openshift:1.4` to the application `catalog` and deploy the uber-jar `catalog-1.0-SNAPSHOT.jar`:

~~~shell
$ odo create redhat-openjdk18-openshift:1.4 service --app catalog \
--binary labs/catalog-spring-boot/target/catalog-1.0-SNAPSHOT.jar
 ✓   Checking component
 ✓   Checking component version
 ✓   Creating component service
 OK  Component 'service' was created and ports 8080/TCP,8443/TCP,8778/TCP were opened
 OK  Component 'service' is now set as active component
To push source code to the component run 'odo push'
~~~

![Catalog Service Component]({% image_path springboot-catalog-component.png %}){:width="500px"}

#### Pushing your source code

Now that the component is running, push our initial source code:

~~~shell
$ odo push service --app catalog
Pushing changes to component: service
 ✓   Waiting for pod to start
 ✓   Copying files to pod
 ✓   Building component
 OK  Changes successfully pushed to component: service
~~~

The jar file has now been pushed to the container, and the process in that container restarted.

#### Creating an OpenShift route

To access to the service outside the cluster, create an external URL (an Openshift Route) for the `Catalog` application:

~~~shell
$ odo url create --app catalog --component service --port 8080
Adding URL to component: service
 OK  URL created for component: service

service - http://service-catalog-{{COOLSTORE_PROJECT}}.{{APPS_HOSTNAME_SUFFIX}}
~~~

> The route urls in your project would be different from the ones in this lab guide! Use the ones from your project.

Copy the route url for the Catalog service into your browser:

![Catalog Service Root]({% image_path springboot-catalog-service-root.png %}){:width="500s"}

#### Updating Component on Change

Watch for Changes and updating Component on Change. Let's run `odo watch` in a new terminal window.

~~~shell
$ odo watch service --app catalog
Waiting for something to change in /projects/labs/catalog-spring-boot/target/catalog-1.0-SNAPSHOT.jar
~~~

Now that the project is ready, let's get coding and create a domain model, data repository, and a RESTful endpoint to create the Catalog service:

![Catalog RESTful Service]({% image_path springboot-catalog-arch.png %}){:width="640px"}

#### Creating the Domain Model

Create a new Java class named `Product` in the `com.redhat.cloudnative.catalog` package with the below code and 
following fields: `itemId`, `name`, `desc` and `price`

In the project explorer in CodeReady Workspaces, right-click on **catalog-spring-boot > src > main > java > com.redhat.cloudnative.catalog** and then on **New > Java Class**. Enter `Product` as the Java class name.


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

Review the `Product` domain model and note the JPA annotations on this class. `@Entity` marks the class as a JPA entity, `@Table` customizes the table creation process by defining a table name and database constraint and `@Id` marks the primary key for the table

Spring Boot configuration is done to a large extent through detecting the intent of the 
developer and automatically adding the required dependencies configurations to make sure it can 
get out of the way and developers can be productive with their code rather than Googling for 
configuration snippets. As an example, configuration database access with JPA is composed of 
the following steps:

1. Adding the `org.springframework.boot:spring-boot-starter-data-jpa` dependency to `pom.xml` 
2. Adding the database driver (e.g. `org.postgresql:postgresql`) to `pom.xml`
3. Adding database connection details in `src/main/resources/project-default.yml`

Edit the `pom.xml` file and add the `org.springframework.boot:spring-boot-starter-data-jpa` dependency to enable JPA:

~~~xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
~~~

Note that the configurations uses `src/main/resources/load.sql` to import initial data into the database.

Examine `src/main/resources/application.properties` to see the database connection details. 
An in-memory H2 database is used in this lab for local development and in the following 
labs will be replaced with a PostgreSQL database. Be patient! More on that later.

#### Creating a Data Repository

Spring Data repository abstraction simplifies dealing with data models in Spring applications by reducing the amount of boilerplate code required to implement data access layers for various persistence stores. [Repository and its sub-interfaces](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.core-concepts) are the central concept in Spring Data which is a marker interface to provide data manipulation functionality for the entity class that is being managed. When the application starts, Spring finds all interfaces marked as repositories and for each interface found, the infrastructure configures the required persistent technologies and provides an implementation for the repository interface.

Create a new Java interface named `ProductRepository` in `com.redhat.cloudnative.catalog` package and extend [CrudRepository](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html) interface in order to indicate to Spring that you want to expose a complete set of methods to manipulate the entity.

In the project explorer in CodeReady Workspaces, right-click on **catalog-spring-boot > src > main > java > com.redhat.cloudnative.catalog** and then on **New > Java Class** and paste the following code:

~~~java
package com.redhat.cloudnative.catalog;

import org.springframework.data.repository.CrudRepository;

public interface ProductRepository extends CrudRepository<Product, String> {
}
~~~

That's it! Now that you have a domain model and a repository to retrieve the domain mode, let's create a 
RESTful service that returns the list of products.

#### Creating a RESTful Service

Spring Boot uses Spring Web MVC as the default RESTful stack in Spring applications. Create a new Java class named `CatalogController` in `com.redhat.cloudnative.catalog` package with the following content by right-clicking on **catalog-spring-boot > src > main > java > com.redhat.cloudnative.catalog** and then clicking on **New > Java Class**:

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

The above REST services defines an endpoint that is accessible via `HTTP GET` at `/api/catalog`. Notice the `repository` field on the controller class which is used to retrieve the list of products. Spring Boot automatically provides an implementation for `ProductRepository` at runtime and 
[injects it into the controller using the `@Autowire` annotation](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-spring-beans-and-dependency-injection.html).

In CodeReady Workspaces, click on **catalog-spring-boot** project in the project explorer, and then click on Commands Palette and click on **BUILD > build**

![Maven Build]({% image_path  codeready-command-build.png %}){:width="200px"}

Once successfully built, your new version of the jar will be pushed automatically into the Catalog Component thanks to the `odo watch` command. You should see following logs in the Terminal where you ran the `odo watch` command.

~~~shell
File /projects/labs/catalog-spring-boot/target/catalog-1.0-SNAPSHOT.jar changed
Pushing files...
 ✓   Waiting for pod to start
 ✓   Copying files to pod
 ✓   Building component
Waiting for something to change in /projects/labs/catalog-spring-boot/target/catalog-1.0-SNAPSHOT.jar
~~~

Now, you can access the Catalog REST API. Let’s test it out using `curl` in a new terminal window:

~~~shell
$ odo url list --component service --app catalog
NAME        URL                                                              PORT
service     http://service-catalog-{{COOLSTORE_PROJECT}}.{{APPS_HOSTNAME_SUFFIX}}      8080
$ curl http://service-catalog-{{COOLSTORE_PROJECT}}.{{APPS_HOSTNAME_SUFFIX}}/api/catalog

[{"itemId":"329299","name":"Red Fedora","desc":"Official Red Hat Fedora","price":34.99},...]
~~~

Well done! You are ready to move on to the next lab.

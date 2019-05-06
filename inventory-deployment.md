## Deploying the Inventory Service

*5 MINUTES PRACTICE*

In this lab you will learn about building and deploying a microservice based on Java/Maven on OpenShift. 
The **Inventory Service** is part of the Cool Store architecture to provide the quantity (hence the availability too) of a given product by ID.

![CoolStore Architecture]({% image_path coolstore-arch-inventory-thorntail.png %}){:width="400px"}

#### What is Thorntail?

Java EE applications are traditionally created as an *ear* or *war* archive including all 
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

The **inventory-thorntail** project has the following structure which shows the components of 
the Thorntail project laid out in different subdirectories according to Maven best practices:

![Inventory Project]({% image_path thorntail-inventory-project.png %}){:width="300px"}

#### Deploy Inventory Service on OpenShift

Now it’s time to build and deploy our service on OpenShift. 

OpenShift [Source-to-Image (S2I)]({{OPENSHIFT_DOCS_BASE}}/architecture/core_concepts/builds_and_image_streams.html#source-build) 
feature can be used to build a container image from a git repository. OpenShift S2I uses the [supported OpenJDK container image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift) to build the final container image of the 
**Inventory Service** by building the Thorntail uber-jar from source code (build strategy **'Source'**), using Maven, to the OpenShift platform.

To build and deploy the **Inventory Service** on OpenShift using the *fabric8* maven plugin, 
which is already configured in CodeReady Workspaces, `right click on inventory-thorntail` project in the project explorer then, `click on Commands > Deploy > fabric8:deploy`

![Fabric8 Deploy]({% image_path codeready-commands-deploy.png %}){:width="600px"}

`fabric8:deploy` will cause the following to happen:

* The Inventory uber-jar is built using Thorntail
* A container image is built on OpenShift containing the Inventory uber-jar and JDK
* All necessary objects are created within the OpenShift project to deploy the Inventory service

Once this completes, your project should be up and running. You can see the expose DNS url for the **Inventory Service** in the [OpenShift Web Console]({{OPENSHIFT_CONSOLE_URL}}) or using OpenShift CLI.

~~~shell
$ oc get routes

NAME        HOST/PORT                                       PATH        SERVICES        PORT        TERMINATION   
inventory   inventory-{{COOLSTORE_PROJECT}}.{{APPS_HOSTNAME_SUFFIX}}                    inventory       8080        None
~~~

> The route urls in your project would be different from the ones in this lab guide!

`Click on the 'Inventory Route'` in the [OpenShift Web Console]({{OPENSHIFT_CONSOLE_URL}}).

![Inventory Service]({% image_path inventory-service.png %}){:width="500px"}

Then `click on 'Test it'`. You should have the following output:

~~~shell
{"itemId":"329299","quantity":35}
~~~

Well done! You are ready to move on to the next lab.

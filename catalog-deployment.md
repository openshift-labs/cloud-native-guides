## Deploying the Catalog Service

*5 MINUTES PRACTICE*

In this lab you will learn about building and deploying a microservice based on Java/Maven on OpenShift. 
The **Catalog Service** is part of the Cool Store architecture to provide a list of products for the CoolStore online shop.

![CoolStore Architecture]({% image_path coolstore-arch-catalog-spring-boot.png %}){:width="400px"}

#### What is Spring Boot?

Spring Boot is an opinionated framework that makes it easy to create stand-alone Spring based 
applications with embedded web containers such as Tomcat (or JBoss Web Server), Jetty and Undertow 
that you can run directly on the JVM using `java -jar`. Spring Boot also allows producing a war 
file that can be deployed on stand-alone web containers.

The opinionated approach means many choices about Spring platform and third-party libraries 
are already made by Spring Boot so that you can get started with minimum effort and configuration.

#### Spring Boot Maven Project 

The **catalog-spring-boot** project has the following structure which shows the components of 
the Spring Boot project laid out in different subdirectories according to Maven best practices:


Once loaded, you should see the following files and be able to navigate amongst the files. The 
components of the Spring Boot project are laid out in different subdirectories according to Maven best practices:

![Catalog Project]({% image_path springboot-catalog-project.png %}){:width="300px"}

#### Deploy Catalog Service on OpenShift

Now it’s time to build and deploy our service on OpenShift. 

OpenShift [Source-to-Image (S2I)]({{OPENSHIFT_DOCS_BASE}}/architecture/core_concepts/builds_and_image_streams.html#source-build) 
feature can be used to build a container image from your project. OpenShift 
S2I uses the [supported OpenJDK container image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift) to build the final container image 
of the **Catalog Service** by uploading the Spring Boot uber-jar from the **target/** 
folder to the OpenShift platform. 

Maven projects can use the [Fabric8 Maven Plugin](https://maven.fabric8.io) in order to use OpenShift S2I for building 
the container image of the application from within the project. This maven plugin is a Kubernetes/OpenShift client 
able to communicate with the OpenShift platform using the REST endpoints in order to issue the commands 
allowing to build a project, deploy it and finally launch a docker process as a pod.

To build and deploy the **Catalog Service** on OpenShift using the *fabric8* maven plugin, 
which is already configured in CodeReady Workspaces, `right click on catalog-spring-boot` project in the project explorer then, `click on Commands > Deploy > fabric8:deploy`

![Fabric8 Deploy]({% image_path codeready-commands-deploy.png %}){:width="600px"}

`fabric8:deploy` will cause the following to happen:

* The Catalog uber-jar is built using Spring Boot
* A container image is built on OpenShift containing the Catalog uber-jar and JDK
* All necessary objects are created within the OpenShift project to deploy the Catalog service

Once this completes, your project should be up and running. You can see the expose DNS url for the **Catalog Service** in the [OpenShift Web Console]({{OPENSHIFT_CONSOLE_URL}}) or using OpenShift CLI.

~~~shell
$ oc get routes

NAME        HOST/PORT                                       PATH        SERVICES        PORT        TERMINATION   
catalog     catalog-{{COOLSTORE_PROJECT}}.{{APPS_HOSTNAME_SUFFIX}}                      catalog         8080        None
inventory   inventory-{{COOLSTORE_PROJECT}}.{{APPS_HOSTNAME_SUFFIX}}                    inventory       8080        None
~~~

> The route urls in your project would be different from the ones in this lab guide!

`Click on the Catalog Route` in the [OpenShift Web Console]({{OPENSHIFT_CONSOLE_URL}}).

![Catalog Service]({% image_path catalog-service.png %}){:width="500px"}

Then `click on 'Test it'`. You should have the following output:

~~~shell
[{"itemId":"329299","name":"Red Fedora","desc":"Official Red Hat Fedora","price":34.99},...]
~~~

Well done! You are ready to move on to the next lab.

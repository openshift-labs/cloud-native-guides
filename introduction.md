## Introduction 

*2 MINUTES PRACTICE*

In this workshop you will learn how to develop and deploy a microservices based application. 

The overall architecture of the application that you will deploy is the following:

![API Gateway Pattern]({% image_path coolstore-arch.png %}){:width="400px"}

During the various steps of the the workshop you will use CodeReady Workspaces, an online IDE that is running on Red Hat OpenShift to write, test and deploy:

* **Catalog Service** exposes using a REST API content of a catalog stored in a relational database
* **Inventory Service** exposes using a REST API the inventory stored in a relational database
* **Gateway Service** calls the **Catalog Service** and **Inventory Service** in an efficient way
* **WebUI Service** calls **Gateway Service** to retrieve all the informations.

In addition to the application code, you will learn how to deploy the various services to OpenShift and how to use it to route the trafic to these services and monitor them.

You will also have the opportunity to look at some optional steps such as debugging, continuous delivery, externalized configuration and more.

Let's start the workshop with the discovery of OpenShift and CodeReady Workspaces.

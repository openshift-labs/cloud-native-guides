## A/B Testing with Service Mesh

*20 MINUTES PRACTICE*


[A/B testing](https://en.wikipedia.org/wiki/A/B_testing) allows running multiple versions of a functionality in parralel; and using analytics of the user behavior it is possible to determine which version is the best. 
It is also possible to launch the new features only for a small set of users, to prepare the general avalability of a new feature. 

In this lab you will see how you can use Site Mesh to do some A/B testing using and route traffic between 2 versions of the Catalog service.

#### Deploying the new Catalog service

A new ***Catalog Service*** has been created, this service is developed in [Golang](https://golang.org/) and available in the following repository:

<{{CATALOG_GO_GIT_REPO}}>

For the rest of the lab we will name this service ***catalog-v2***. This service use the same business logic except that all product descriptions are returned in uppercase.

Let's deploy the service directly from the git repository using the `oc new-app` command.

In the terminal window type the following command:

~~~shell
$ oc new-app {{LABS_GIT_REPO}} \
    --strategy=docker \
    --context-dir=catalog-go \
    --name=catalog-v2 \
    --labels app=catalog,group=com.redhat.cloudnative,provider=fabric8,version=2.0
 
~~~

*Note: To simplify the lab, we use the same labels for catalog-v2 and v1, since they are used for the service routing.*

Service Mesh will be used to route the traffic between the catalog service v1 and v2, so you have too add the Istio sidecar to the catalog v2 using the following command:

~~~shell
$ oc patch dc/catalog-v2 --patch \
  '{"spec": {"template": {"metadata": {"annotations": {"sidecar.istio.io/inject": "true"}}}}}'
~~~

Let's now create the `Destination Rule` resource using the following command:

~~~shell
$ cat << EOF | oc create -f -
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: catalog
spec:
  host: catalog
  subsets:
  - labels:
      version: "1.0-SNAPSHOT"
    name: "version-springboot"
  - labels:
      version: "2.0"
    name: "version-go"
EOF
~~~

Destination Rule defines policies that apply to traffic intended for a service after routing has occurred. 

If you want to learn more about Istio Destination Rules you can cool to the [documentation](https://istio.io/docs/reference/config/networking/v1alpha3/destination-rule/).


The last step is to define the rules to distribute the traffic between the services. For this you have to create an Istio virtual service using the following command:


~~~shell
$ cat << EOF | oc create -f -
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
    - catalog
  http:
  - route:
    - destination:
        host: catalog
        subset: "version-springboot"
      weight: 50
    - destination:
        host: catalog
        subset: "version-go"
      weight: 50
EOF
~~~

Using the last command you are asking Istio to route the traffic between Catalog v1 (Spring Boot) and v2 (Go) equally.

#### Running the application.

Let's now see the A/B testing with Site Mesh in action, for this you can run the `./labs/runGatewayService.sh` that call the REST API multiple times.

Open a new terminal in CodeReady Workspaces, using the menu **Run > Terminal**, and run the following commands:

~~~shell

$ cd ./labs/scripts

$ chmod +x runGatewayService.sh

$ ./runGatewayService.sh {{COOLSTORE_PROJECT}}

~~~~

The shell script will print the version of the service that is used (v1 or v2).

You can also go to the Web interface and refresh the page to see that the site uses sometimes product description in uppercare (v2) or lowercase (v1).

Go to Kiali to see the traffic distribution between Catalog v1 and v2.

Click on the `Graph` link in the left navigation and enter the following configuration:

* Namespace: `coolstore`
* Display: `check Traffic Animation`
* Edge Label: `Traffic rate per second`
* Fetching: `Last 5 min`

![Kiali- Graph]({% image_path ab-testing-kiali-001.png %}){:width="700px"}


This page shows a graph with all the microservices and the distribution of the traffic between them.


You can now easily change the traffic rules, for example to switch all the traffic to the catalog-v2. For this change the weight in the Istio VirtualService configuration.

For this go back to the first terminal and run the following command that will update the Istio VirtualService configuration to route 100% of the traffic to `catalog-v2`.

~~~shell
$ cat << EOF | oc replace -f -
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
    - catalog
  http:
  - route:
    - destination:
        host: catalog
        subset: "version-springboot"
      weight: 0
    - destination:
        host: catalog
        subset: "version-go"
      weight: 100
EOF
~~~

You should automatically see some change in the Kiali charts, the console and the Web application.


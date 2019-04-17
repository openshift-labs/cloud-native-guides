## A/B Testing with Service Mesh

*10 MINUTES PRACTICE*


[A/B testing](https://en.wikipedia.org/wiki/A/B_testing) allows running multiple versions of a functionality in parallel; and using analytics of the user behavior it is possible to determine which version is the best. 
It is also possible to launch the new features only for a small set of users, to prepare the general avalability of a new feature. 

In this lab you will see how you can use Site Mesh to do some A/B testing using and route traffic between 2 versions of the Catalog service.

#### Deploying the new Catalog service

A new ***Catalog Service v2*** has been created, this service is developed in [Golang](https://golang.org/) and available in the following repository:

<{{CATALOG_GO_GIT_REPO}}>

This service use the same business logic except that all product descriptions are returned in uppercase.

Let's deploy the service directly from the git repository using the `oc new-app` command.

In the terminal window type the following command:

~~~shell
$ oc new-app {{LABS_GIT_REPO}} \
    --strategy=docker \
    --context-dir=catalog-go \
    --name=catalog-v2 \
    --labels app=catalog,group=com.redhat.cloudnative,provider=fabric8,version=2.0
 
~~~

> **Note**: To simplify the lab, we use the same labels for ***catalog*** and ***catalog-v2***, since they are used for the service routing.

Service Mesh will be used to route the traffic between the catalog service v1 and v2, so you have to add the Istio sidecar to the ***Catalog Service v2*** using the following command:

~~~shell
$ oc patch dc/catalog-v2 --patch \
  '{"spec": {"template": {"metadata": {"annotations": {"sidecar.istio.io/inject": "true"}}}}}'
~~~

To confirm that the application is successfully deployed, run this command:

~~~shell
$ oc get pods -lapp=catalog,deploymentconfig=catalog-v2
NAME                 READY     STATUS    RESTARTS   AGE
catalog-v2-2-7zsxb   2/2       Running   0          1m
~~~

The status should be **Running** and there should be **2/2** pods in the **Ready** column.
Wait few seconds that the application restarts.


#### Enabling A/B Testing

[A/B Testing](https://en.wikipedia.org/wiki/A/B_testing) allows to run in parallel two versions of an application with one single variant (usually visual) and to collect metrics in order to determine the variant with the best effect of the user behavior.

The implementation of such procedure is one are the advantages coming with OpenShift Service Mesh.

For this lab, you want to answer the following question: **Do the product descriptions written in uppercase increase sales rate?**

Let's now create the ***Destination Rule*** resource.

* A ***Destination Rule*** defines policies that apply to traffic intended for a service after routing has occurred. These rules specify configuration for load balancing, connection pool size from the sidecar, and outlier detection settings to detect and evict unhealthy hosts from the load balancing pool.

In the Terminal window, issue the following command:

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

Now you have created a ***Destination Rule*** for ***Catalog Service*** and ***Catalog Service v2***.

The last step is to define the rules to distribute the traffic between the services. 

* A **VirtualService** defines a set of traffic routing rules to apply when a host is addressed. Each routing rule defines matching criteria for traffic of a specific protocol. If the traffic is matched, then it is sent to a named destination service (or subset/version of it) defined in the registry.

In the Terminal window, issue the following command:

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

Doing so, you route **50%** of the **HTTP traffic** to pods of the ***Catalog Service*** *(subset "version-springboot" ie label "version: 1.0-SNAPSHOT")* and the **50%** remaining to pods of the ***Catalog Service v2*** *(subset "version-go" ie label "version: 2.0")*.

#### Generate HTTP traffic.

Let's now see the A/B testing with Site Mesh in action.
First, we need to generate HTTP traffic by sending several requests to the ***Gateway Service*** from the ***Istio Gateway***

In CodeReady Workspaces, click on ***Commands Palette*** and click on **RUN > runGatewayService**
![Commands Palette - RunGatewayService]({% image_path  codeready-command-run-gateway-service.png %}){:width="600px"}

You likely see *'Gateway => Catalog Spring Boot (v1)'* or *'Gateway => Catalog GoLang (v2)'*

![Terminal - RunGatewayService]({% image_path  codeready-run-gateway-50-50.png %}){:width="400px"}

> You can also go to the Web interface and refresh the page to see that product descriptions is sometimes in uppercase (v2) or not (v1).

Go to Kiali to see the traffic distribution between Catalog v1 and v2.

From the [Kiali Console]({{ KIALI_URL }}) *(please make sure to replace **infrax** with your dedicated project)*, `click on the 'Graph' link` in the left navigation and enter the following configuration:

 * Namespace: **{{COOLSTORE_PROJECT}}**
 * Display: **check 'Traffic Animation'**
 * Edge Label: **Requests percent of total**
 * Fetching: **Last 5 min**

![Kiali- Graph]({% image_path kiali-abtesting-50-50.png %}){:width="700px"}

You can see that the traffic between the two version of the ***Catalog*** is shared equitably (at least very very close). 

After one week trial, you have collected enough information to confirm that product descriptions in uppercase do increate sales rates. So you will route all the traffic to ***Catalog Service v2***. Go back to the Terminal and run the following command:

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

Now, you likely see only *'Gateway => Catalog GoLang (v2)'* in the *'runGatewayService'* terminal.

![Terminal - RunGatewayService]({% image_path  codeready-run-gateway-100.png %}){:width="600px"}

And from [Kiali Console]({{ KIALI_URL }}) *(please make sure to replace **infrax** with your dedicated project)*, you can visualize that **100%** of the traffic is switching gradually to ***Catalog Service v2***.

![Kiali- Graph]({% image_path kiali-abtesting-100.png %}){:width="700px"}

That's all for this lab! You are ready to move on to the next lab.

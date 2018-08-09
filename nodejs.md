## Web UI with Node.js and AngularJS 

In this lab you will learn about Node.js and will deploy the Node.js and Angular-based 
web front+end for the CoolStore online shop which uses the API Gateway services you deployed 
in previous labs. 

![API Gateway Pattern]({% image_path coolstore-arch.png %}){:width="400px"}

#### What is Node.js?

Node.js is an open source, cross-platform runtime environment for developing server-side 
applications using JavaScript. Node.js has an event-driven architecture capable of 
non-blocking I/O. These design choices aim to optimize throughput and scalability in 
Web applications with many input/output operations, as well as for real-time web applications.

Node.js non-blocking architecture allows applications to process large number of 
requests (tens of thousands) using a single thread which makes it desirable choice for building 
scalable web applications.

#### Deploy Web UI on OpenShift

The Web UI is built using Node.js for server-side JavaScript and AngularJS for client-side 
JavaScript. Let's deploy it on OpenShift using the certified Node.js container image available 
in OpenShift. 

In the previous labs, you used the OpenShift 
[Source-to-Image (S2I)]({{OPENSHIFT_DOCS_BASE}}/architecture/core_concepts/builds_and_image_streams.html#source-build) 
feature via the [Fabric8 Maven Plugin](https://maven.fabric8.io) to build a container image from the 
source code on your laptop. In this lab, you will still use S2I but instead instruct OpenShift 
to obtain the application code directly from the source repository and build and deploy a 
container image of it.

The source code for the the Node.js Web front-end is available in this Git repository: 

<{{WEB_NODEJS_GIT_REPO}}>

Use the OpenShift CLI command to create a new build and deployment for the Web component:

> Feeling adventurous? Build and deploy the Web front-end via the OpenShift Web Console 
> instead. To give you a hint, start by clicking on **Add to project** within the 
> **{{COOLSTORE_PROJECT}}** project and pick **JavaScript** and then **Node.js** in the service 
> catalog. Don't forget to click on **advanced options** and set **Context Dir** to `web-nodejs` 
> which is the sub-folder of the Git repository where the source code for Web resides.

~~~shell
$ oc new-app nodejs:8~{{LABS_GIT_REPO}} \
        --context-dir=web-nodejs \
        --name=web 
~~~

The `--context-dir` option specifies the sub-directly of the Git repository which contains 
the source code for the application to be built and deployed. The `--labels` allows 
assigning arbitrary key-value labels to the application objects in order to make it easier to 
find them later on when you have many applications in the same project.

A build gets created and starts building the Node.js Web UI container image. You can see the build 
logs using OpenShift Web Console or OpenShift CLI:

~~~shell
$ oc logs -f bc/web
~~~

The `-f` option is to follow the logs as the build progresses. After the building the Node.s Web UI 
completes, it gets pushed into the internal image registry in OpenShift and then deployed within 
your project.

In order to access the Web UI from outside (e.g. from a browser), it needs to get added to the load 
balancer. Run the following command to add the Web UI service to the built-in HAProxy load balancer 
in OpenShift.

~~~shell
$ oc expose svc/web
$ oc get route web
~~~

Point your browser at the Web UI route url. You should be able to see the CoolStore with all 
products and their inventory status.

![CoolStore Shop]({% image_path coolstore-web.png %}){:width="840px"}

Well done! You are ready to move on to the next lab.

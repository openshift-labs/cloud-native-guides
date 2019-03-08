## Web UI with Node.js and AngularJS 

In this lab you will learn about Node.js and will deploy the Node.js and Angular-based 
web frontend for the CoolStore online shop which uses the API Gateway services you deployed 
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

#### Creating an Openshift Application

An application is an umbrella of components that work together to implement the overall application. OpenShift helps organize these modular applications with a concept called, appropriately enough, the application. An OpenShift application represents all of an app's components in a logical management unit.

First, create an application called `web` to work with:

~~~shell
$ odo app create web
Creating application: web in project: {{COOLSTORE_PROJECT}}
Switched to application: web in project: {{COOLSTORE_PROJECT}}
~~~

You can verify that the new application is created with the following commands:

~~~shell
$ odo app list
The project '{{COOLSTORE_PROJECT}}' has the following applications:
ACTIVE     NAME
           inventory
           catalog
           gateway
*          web
~~~

#### Creating a Service Component from Source Code

Add a component named `ui` of type `nodejs:8` to the application `web` and deploy the initial source code:

~~~shell
$ odo create nodejs:8 ui --app web --local labs/web-nodejs --env COOLSTORE_GW_ENDPOINT="service-gateway-{{COOLSTORE_PROJECT}}"
 ✓   Checking component
 ✓   Checking component version
 ✓   Creating component ui
 OK  Component 'ui' was created and port 8080/TCP was opened
 OK  Component 'ui' is now set as active component
To push source code to the component run 'odo push'
~~~

![Web ui Component]({% image_path nodejs-webui-component.png %}){:width="500px"}

#### Pushing your source code

Now that the component is running, push our initial source code:

~~~shell
$ odo push ui --app web
Pushing changes to component: ui
 ✓   Waiting for pod to start
 ✓   Copying files to pod
 ✓   Building component
 OK  Changes successfully pushed to component: ui
~~~

The source code has now been pushed to the container, and the process in that container restarted.

#### Creating an OpenShift route

To access to the user interface outside the cluster, create an external URL (an Openshift Route) for the `Web` application:

~~~shell
$ odo url create --app web --component ui --port 8080
Adding URL to component: ui
 OK  URL created for component: ui

ui - http://ui-web-{{COOLSTORE_PROJECT}}.{{APPS_HOSTNAME_SUFFIX}}
~~~

> The route urls in your project would be different from the ones in this lab guide! Use the ones from your project.

Point your browser at the Web UI route url. You should be able to see the CoolStore with all 
products and their inventory status.

![CoolStore Shop]({% image_path coolstore-web.png %}){:width="500px"}


#### Updating Component on Change

Watch for Changes and updating Component on Change. Let's run `odo watch` in a new terminal window.

~~~shell
$ odo watch ui --app web
Waiting for something to change in /projects/labs/web-nodejs
~~~


#### Changing the Background Color

Next, let's make a change to the user interface that will be obvious in the UI.

First, open `labs/web-nodejs/app/css/coolstore.css`, which contains the CSS stylesheet for the CoolStore app.

Add the following CSS to turn the header bar background to Red Hat red:

~~~css
.navbar-header {
    background: #CC0000
}
~~~

Once saved, the modified file `coolstore.css` will be detected and will be pushed automatically into the UI Component thanks to the `odo watch` command. You should see following logs in the Terminal where you ran the `odo watch` command.

~~~shell
File /projects/labs/web-nodejs/app/css/coolstore.css changed
Pushing files...
 ✓   Waiting for pod to start
 ✓   Copying files to pod
 ✓   Building component
Waiting for something to change in /projects/labs/web-nodejs
~~~

Reload the Coolstore webpage and you should now see the red header.

![CoolStore Shop]({% image_path coolstore-web-red.png %}){:width="500px"}

Well done! You are ready to move on to the next lab.

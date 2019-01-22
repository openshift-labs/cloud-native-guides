## Getting Started with OpenShift

In this lab you will get familiar with the OpenShift CLI and OpenShift Web Console 
and get ready for the Cloud Native Roadshow labs.

For completing the following labs, you can either use your own workstation or as an 
alternative, CodeReady Workspaces Web IDE. The advantage of your own workstation is that you use the 
environment that you are familiar with while the advantage of CodeReady Workspaces is that all 
tools needed (Maven, Git, OpenShift CLI, etc ) are pre-installed in it (not on your workstation!) and all interactions 
takes place within the browser which removes possible internet speed issues and version incompatibilities 
on your workstation.

The choice is yours but whatever you pick, like most things in life, stick with it for all the labs. We 
ourselves are in love with CodeReady Workspaces and highly recommend it.

## Setup Your Workspace on CodeReady Workspaces

Follow these instructions to setup the development environment on CodeReady Workspaces. 

You might be familiar with the Eclipse IDE which is one of the most popular IDEs for Java and other programming languages. Built on the open [Eclipse Che](https://www.eclipse.org/che/) project, [CodeReady Workspaces](https://developers.redhat.com/products/codeready-workspaces/overview/) is the next-generation Eclipse IDE which provides developer workspaces, which include all the tools and the dependencies that are needed to code, build, test, run, and debug applications. This full-featured web-based IDE runs in an OpenShift cluster hosted on-premises or in the cloud and eliminates the need to install anything on a local machine.You have a CodeReady Workspaces instance deployed on your OpenShift cluster which you will use during these labs.

Go to the [CodeReady Workspaces url]({{ CODEREADY_WORKSPACES_URL }}) in order to configure your development workspace: {{ CODEREADY_WORKSPACES_URL }}

#### Registering to CodeReady Workspaces
First, you need to register as a user. Register and choose the same username and password as 
your OpenShift credentials.

![CodeReady Workspaces - Register]({% image_path codeready-register.png %}){:width="500px"}

#### Creating a Workspace
Log into CodeReady Workspaces with your user. You can now create your workspace based on a stack. A 
stack is a template of workspace configuration. For example, it includes the programming language and tools needed
in your workspace. Stacks make it possible to recreate identical workspaces with all the tools and needed configuration
on-demand. 

For this lab, click on the **Java Cloud-Native** stack and then on the **Create** button. 

![CodeReady Workspaces - Workspace]({% image_path codeready-create-workspace.png %}){:width="1000px"}

Click on **OPEN** to open and to start the workspace.

![CodeReady Workspaces - Workspace]({% image_path codeready-start-workspace.png %}){:width="1000px"}

It takes a little while for the workspace to be ready. When it's ready, you will see a fully functional CodeReady Workspaces IDE running in your browser.

![CodeReady Workspaces - Workspace]({% image_path codeready-workspace.png %}){:width="1000px"}

#### Importing the lab project
Now you can import the project skeletons into your workspace.

In the project explorer pane, click on the **Import Project...** and enter the following:

  * Type: `ZIP`
  * URL: `{{LABS_DOWNLOAD_URL}}`
  * Name: `labs`
  * Check **Skip the root folder of the archive**

![CodeReady Workspaces - Import Project]({% image_path codeready-import.png %}){:width="500px"}

Click on **Import**. Make sure you choose the **Blank** project configuration since the zip file contains multiple 
project skeletons. Click on **Save**

![CodeReady Workspaces - Import Project]({% image_path codeready-import-save.png %}){:width="500px"}

#### Converting your project skeletons
The projects are imported now into your workspace and is visible in the project explorer.

CodeReady Workspaces is a full featured IDE and provides language specific capabilities for various project types. In order to 
enable these capabilities, let's convert the imported project skeletons to Maven projects. 

In the project explorer, right-click on **catalog-spring-boot** and then click on **Convert to Project**.

![CodeReady Workspaces - Convert to Project]({% image_path codeready-convert.png %}){:width="500px"}

Choose **Maven** from the project configurations and then click on **Save**

![CodeReady Workspaces - Convert to Project]({% image_path codeready-maven.png %}){:width="500px"}

> Repeat the above for **inventory-thorntail** and **gateway-vertx** projects.

> Convert the **web-nodejs** project into **NodeJS**.

> The **Terminal** window in CodeReady Workspaces. For the rest of these labs, anytime you need to run a command in a terminal, you can use the CodeReady Workspaces **Terminal** window.
> ![CodeReady Workspaces - Terminal]({% image_path codeready-terminal.png %}){:width="700px"}

## Explore OpenShift with OpenShift DO

[OpenShift Do (Odo)](https://openshiftdo.org/) is a CLI tool for developers who are writing, building, and deploying applications on OpenShift. With Odo, developers get an opinionated CLI tool that supports fast, iterative development which abstracts away Kubernetes and OpenShift concepts, thus allowing them to focus on what's most important to them: **CODE**.

In order to login, we will use the `odo` command and then specify the server that we
want to authenticate to.

Issue the following command in CodeReady Workspaces terminal and replace `{{OPENSHIFT_CONSOLE_URL}}` 
with your OpenShift Web Console url. 

~~~shell
$ odo login {{OPENSHIFT_CONSOLE_URL}}
~~~

You may see the following output:

~~~shell
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n):
~~~

Enter in `y` to use a potentially insecure connection.  The reason you received
this message is because we are using a self-signed certificate for this
workshop, but we did not provide you with the CA certificate that was generated
by OpenShift. In a real-world scenario, either OpenShift's certificate would be
signed by a standard CA (eg: Thawte, Verisign, StartSSL, etc.) or signed by a
corporate-standard CA that you already have installed on your system.

Enter the username and password provided to you by the instructor

Congratulations, you are now authenticated to the OpenShift server.

[Projects]({{OPENSHIFT_DOCS_BASE}}/architecture/core_concepts/projects_and_users.html#projects) 
are a top level concept to help you organize your deployments. An
OpenShift project allows a community of users (or a user) to organize and manage
their content in isolation from other communities. Each project has its own
resources, policies (who can or cannot perform actions), and constraints (quotas
and limits on resources, etc). Projects act as a "wrapper" around all the
application services and endpoints you (or your teams) are using for your work.

For this lab, let's create a project that you will use in the following labs for 
deploying your applications. 

> Make sure to follow your instructor guidance on the project names in order to 
> have a unique project name for yourself e.g. appending your username to the project name

~~~shell
$ odo project create {{COOLSTORE_PROJECT}}
OK  New project created and now using project : {{COOLSTORE_PROJECT}}
~~~

OpenShift ships with a web-based console that will allow users to
perform various tasks via a browser.  To get a feel for how the web console
works, open your browser and go to the OpenShift Web Console.


The first screen you will see is the authentication screen. Enter your username and password and 
then log in. After you have authenticated to the web console, you will be presented with a
list of projects that your user has permission to work with. 

Click on the **{{COOLSTORE_PROJECT}}** project to be taken to the project overview page
which will list all of the routes, services, deployments, and pods that you have
running as part of your project. There's nothing there now, but that's about to
change.

Due to security reasons, by default, containers are not allowed to access to the OpenShift REST API. We need to grant them permission in order to use Service and Config Map discovery features later.

> Make sure to replace the project name with your own unique project name

~~~shell
$ oc policy add-role-to-user view -n {{COOLSTORE_PROJECT}} -z default
~~~ 

Now you are ready to get started with the labs!

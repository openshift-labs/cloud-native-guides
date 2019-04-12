## Getting Started with OpenShift

*5 MINUTES PRACTICE*

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

`Go to the` [CodeReady Workspaces url]({{ CODEREADY_WORKSPACES_URL }}) in order to configure your development workspace.,

#### Logging in to CodeReady Workspaces
First, you need to log in as `{{OPENSHIFT_USER}}/{{OPENSHIFT_PASWORD}}`

![CodeReady Workspaces - Log in]({% image_path codeready-login.png %}){:width="500px"}

#### Creating a Workspace
Once logged into CodeReady Workspaces, you can now create your workspace based on a stack. A 
**Stack** is a template of workspace configuration. For example, it includes the programming language and tools needed
in your workspace. Stacks make it possible to recreate identical workspaces with all the tools and needed configuration
on-demand. 

For this lab, `select the Java Cloud-Native stack` and then `click on 'CREATE & OPEN'`. 

![CodeReady Workspaces - Workspace]({% image_path codeready-create-workspace.png %}){:width="1000px"}

It takes a little while for the workspace to be ready. When it's ready, you will see a fully functional CodeReady Workspaces IDE running in your browser.

![CodeReady Workspaces - Workspace]({% image_path codeready-workspace.png %}){:width="1000px"}

#### Importing the lab project
Now you can import the project skeletons into your workspace.

In the project explorer pane, `click on 'Import Project...'` and enter the following:

  * Type: **ZIP**
  * URL: **{{LABS_DOWNLOAD_URL}}**
  * Name: **labs**
  * Check **Skip the root folder of the archive**

![CodeReady Workspaces - Import Project]({% image_path codeready-import.png %}){:width="500px"}

`Click on 'Import'`. Make sure you choose the **Blank** project configuration since the zip file contains multiple 
project skeletons. `Click on 'Save'`

![CodeReady Workspaces - Import Project]({% image_path codeready-import-save.png %}){:width="500px"}

#### Converting your project skeletons
The projects are imported now into your workspace and is visible in the project explorer.

CodeReady Workspaces is a full featured IDE and provides language specific capabilities for various project types. In order to 
enable these capabilities, let's convert the imported project skeletons to Maven projects. 

In the project explorer, `right-click on 'catalog-spring-boot'` project in the project explorer then, `click on 'Convert to Project'`.

![CodeReady Workspaces - Convert to Project]({% image_path codeready-convert.png %}){:width="500px"}

Choose **Maven** from the project configurations and then `click on 'Save'`

![CodeReady Workspaces - Convert to Project]({% image_path codeready-maven.png %}){:width="500px"}

> Repeat the above for **inventory-thorntail** and **gateway-vertx** projects.

> Convert the **web-nodejs** project into **NodeJS**.

> The **Terminal** window in CodeReady Workspaces. For the rest of these labs, anytime you need to run a command in a terminal, you can use the CodeReady Workspaces **Terminal** window.
> ![CodeReady Workspaces - Terminal]({% image_path codeready-terminal.png %}){:width="700px"}

## Explore OpenShift with OpenShift CLI

In order to login, we will use the `oc` command and then specify the server that we
want to authenticate to.

Issue the following command in CodeReady Workspaces terminal and log in as `{{OPENSHIFT_USER}}/{{OPENSHIFT_PASWORD}}`

~~~shell
$ oc login {{OPENSHIFT_CONSOLE_URL}}
~~~

You may see the following output:

~~~shell
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n):
~~~

Enter in `Y` to use a potentially insecure connection.  The reason you received
this message is because we are using a self-signed certificate for this
workshop, but we did not provide you with the CA certificate that was generated
by OpenShift. In a real-world scenario, either OpenShift's certificate would be
signed by a standard CA (eg: Thawte, Verisign, StartSSL, etc.) or signed by a
corporate-standard CA that you already have installed on your system.

Congratulations, you are now authenticated to the OpenShift server.

[Projects]({{OPENSHIFT_DOCS_BASE}}/architecture/core_concepts/projects_and_users.html#projects) 
are a top level concept to help you organize your deployments. An
OpenShift project allows a community of users (or a user) to organize and manage
their content in isolation from other communities. Each project has its own
resources, policies (who can or cannot perform actions), and constraints (quotas
and limits on resources, etc). Projects act as a "wrapper" around all the
application services and endpoints you (or your teams) are using for your work.

 > Make sure to use your dedicated project {{COOLSTORE_PROJECT}} by running the following command `oc project {{COOLSTORE_PROJECT}}`

OpenShift ships with a web-based console that will allow users to
perform various tasks via a browser.  To get a feel for how the web console
works, open your browser and `go to` [OpenShift Web Console]({{OPENSHIFT_CONSOLE_URL}}).


The first screen you will see is the authentication screen. Enter your username and password (`{{OPENSHIFT_USER}}/{{OPENSHIFT_PASWORD}}`) and 
then log in. After you have authenticated to the web console, you will be presented with a
list of projects that your user has permission to work with. 

`Click on '{{COOLSTORE_PROJECT}}'` project to be taken to the project overview page
which will list all of the routes, services, deployments, and pods that you have
running as part of your project. There's nothing there now, but that's about to
change.

Now you are ready to get started with the labs!

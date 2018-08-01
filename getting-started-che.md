## Getting Started with OpenShift

In this lab you will get familiar with the OpenShift CLI and OpenShift Web Console 
and get ready for the Cloud Native Roadshow labs.

For completing the following labs, you can either use your own workstation or as an 
alternative, Eclipse Che web IDE. The advantage of your own workstation is that you use the 
environment that you are familiar with while the advantage of Eclipse Che is that all 
tools needed (Maven, Git, OpenShift CLI, etc ) are pre-installed in it (not on your workstation!) and all interactions 
takes place within the browser which removes possible internet speed issues and version incompatibilities 
on your workstation.

The choice is yours but whatever you pick, like most things in life, stick with it for all the labs. We 
ourselves are in love with Eclipse Che and highly recommend it.

## Setup Your Workspace on Eclipse Che

Follow these instructions to setup the development environment on Eclipse Che. 

You might be familiar with the Eclipse IDE which is one of the most popular IDEs for Java and other
programming languages. [Eclipse Che](https://www.eclipse.org/che/) is the next-generation Eclipse IDE which is web-based
and gives you a full-featured IDE running in the cloud. You have an Eclipse Che instance deployed on your OpenShift cluster
which you will use during these labs.

Go to the [Eclipse Che url]({{ ECLIPSE_CHE_URL }}) in order to configure your development workspace: {{ ECLIPSE_CHE_URL }}

First, you need to register as a user. Register and choose the same username and password as 
your OpenShift credentials.

![Eclipse Che - Register]({% image_path bootstrap-che-register.png %}){:width="700px"}

Log into Eclipse Che with your user. You can now create your workspace based on a stack. A 
stack is a template of workspace configuration. For example, it includes the programming language and tools needed
in your workspace. Stacks make it possible to recreate identical workspaces with all the tools and needed configuration
on-demand. 

For this lab, click on the **Java Cloud-Native** stack and then on the **Create** button. 

![Eclipse Che Workspace]({% image_path bootstrap-che-create-workspace.png %})

Click on **Open** to open the workspace and then on the **Start** button to start the workspace for use, if it hasn't started automatically.

![Eclipse Che Workspace]({% image_path bootstrap-che-start-workspace.png %})

You can click on the left arrow icon to switch to the wide view:

![Eclipse Che Workspace]({% image_path bootstrap-che-wide.png %}){:width="600px"}

It takes a little while for the workspace to be ready. When it's ready, you will see a fully functional 
Eclipse Che IDE running in your browser.

![Eclipse Che Workspace]({% image_path bootstrap-che-workspace.png %})

Now you can import the project skeletons into your workspace.

In the project explorer pane, click on the **Import Projects...** and enter the following:

  * Type: `ZIP`
  * URL: `{{LABS_DOWNLOAD_URL}}`
  * Name: `labs`
  * Check **Skip the root folder of the archive**

![Eclipse Che - Import Project]({% image_path bootstrap-che-import.png %}){:width="700px"}

Click on **Import**. Make sure you choose the **Blank** project configuration since the zip file contains multiple 
project skeletons. Click on **Save**

![Eclipse Che - Import Project]({% image_path bootstrap-che-import-save.png %}){:width="700px"}

The projects are imported now into your workspace and is visible in the project explorer.

Eclipse Che is a full featured IDE and provides language specific capabilities for various project types. In order to 
enable these capabilities, let's convert the imported project skeletons to a Maven projects. 

In the project explorer, right-click on **catalog-spring-boot** and then click on **Convert to Project**.

![Eclipse Che - Convert to Project]({% image_path bootstrap-che-convert.png %}){:width="600px"}

Choose **Maven** from the project configurations and then click on **Save**

![Eclipse Che - Convert to Project]({% image_path bootstrap-che-maven.png %}){:width="700px"}

Repeat the above for **inventory-wildfly-swarm** and **gateway-vertx** projects.

Note the **Terminal** window in Eclipse Che. For the rest of these labs, anytime you need to run 
a command in a terminal, you can use the Eclipse Che **Terminal** window.

![Eclipse Che - Terminal]({% image_path bootstrap-che-terminal.png %})

## Explore OpenShift with OpenShift CLI

In order to login, we will use the `oc` command and then specify the server that we
want to authenticate to.

Issue the following command in Eclipse Che terminal and replace `{{OPENSHIFT_CONSOLE_URL}}` 
with your OpenShift Web Console url. 

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
$ oc new-project {{COOLSTORE_PROJECT}}

Now using project "{{COOLSTORE_PROJECT}}" on server ...
...
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

Now you are ready to get started with the labs!

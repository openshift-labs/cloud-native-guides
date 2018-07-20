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

## Setup Eclipse Che

Follow these instructions to setup the development environment on Eclipse Che. Note that if you'd rather 
to use your own workstation, you can skip this section and move to **Setup Your Own Workstation**.

You might be familiar with the Eclipse IDE which is one of the most popoular IDEs for Java and other
programming languages. [Eclipse Che](https://www.eclipse.org/che/) is the next-generation Eclipse IDE which is web-based
and gives you a full-featured IDE running in the cloud. You have an Eclipse Che instance deployed on your OpenShift cluster
which you will use during these labs.

Go to the [Eclipse Che url]({{ ECLIPSE_CHE_URL }}) in order to configuration your development workspace: {{ ECLIPSE_CHE_URL }}

A stack is a template of workspace configuration. For example, it includes the programming language and tools needed
in your workspace. Stacks make it possible to recreate identical workspaces with all the tools and needed configuration
on-demand. 

For this lab, click on the **Java with OpenShift CLI** stack and then on the **Create** button. 

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

![Eclipse Che - Convert to Project]({% image_path bootstrap-che-terminal.png %})

## Setup Your Own Workstation

Follow these instructions to setup the development environment on your workstation. Note that if you'd rather 
to use Eclipse Che, you can skip this section and move on to **Explore OpenShift with OpenShift CLI**.

In order to perform the labs on your own workstation, you will need the following installed:

* Java Development Kit 8
* Apache Maven 3.3.x 
* Git client
* A Text Editor (e.g. Atom, Visual Studio Code) or an IDE (JBoss Developer Studio, Eclipse, IntelliJ)

#### OpenShift CLI

OpenShift ships with a feature rich web console as well as command line tools
to provide users with a nice interface to work with applications deployed to the
platform. The OpenShift tools are a single executable written in the Go
programming language and is available for Microsoft Windows, Apple OS X and Linux.

You might already have the OpenShift CLI available on your environment. You can verify 
it by running an `oc` command:

~~~shell
$ oc version
~~~

If the `oc` doesn't exist or you have an older version of the OpenShift CLI, follow 
the next sections to install or update the OpenShift CLI. Otherwise, skip to the 
**Explore OpenShift with OpenShift CLI** section.

#### Download and Install OpenShift CLI on Windows

Download the the OpenShift CLI tool for [Microsoft Windows]({{DOWNLOAD_CLIENT_WINDOWS}})

Once the file has been downloaded, you will need to extract the contents as it
is a compressed archive. I would suggest saving this file to the following
directories:

~~~shell
C:\OpenShift
~~~

In order to extract a zip archive on windows, you will need a zip utility
installed on your system.  With newer versions of windows (greater than XP),
this is provided by the operating system.  Just right click on the downloaded
file using file explorer and select to extract the contents.

Now you can add the OpenShift CLI tools to your PATH. Because changing your PATH 
on windows varies by version of the operating system, we will not list each operating system here.  
However, the general workflow is right click on your computer name inside of the file
 explorer. Select Advanced system settings. I guess changing your PATH is considered 
 an advanced task? :) Click on the advanced tab, and then finally click on Environment variables.
Once the new dialog opens, select the Path variable and add `;C:\OpenShift` at
the end.  For an easy way out, you could always just copy it to C:\Windows or a
directory you know is already on your path. For more detailed instructions:

[Windows XP](https://support.microsoft.com/en-us/kb/310519)

[Windows Vista](http://banagale.com/changing-your-system-path-in-windows-vista.htm)

[Windows 7](http://geekswithblogs.net/renso/archive/2009/10/21/how-to-set-the-windows-path-in-windows-7.aspx)

[Windows 8](http://www.itechtics.com/customize-windows-environment-variables/)

Windows 10 - Follow the directions above.

At this point, we should have the oc tool available for use.  Let's test this
out by printing the version of the oc command:

~~~shell
> oc version
~~~

You should see the OpenShift version.

#### Download and Install OpenShift CLI on Linux

Download the the OpenShift CLI tool for [Linux 64]({{DOWNLOAD_CLIENT_LIN64}})

Once the file has been downloaded, you will need to extract the contents as it
is a compressed archive. I would suggest saving this file to the following
directories:

~~~shell
~/openShift
~~~

Open up a terminal window and change to the directory where you downloaded the
file.  Once you are in the directory, enter in the following command:

~~~shell
$ tar zxvf oc-linux.tar.gz
~~~

The tar.gz file name needs to be replaced by the entire name that was downloaded in the previous step.

Now you can add the OpenShift CLI tools to your PATH.

~~~shell
$ export PATH=$PATH:~/openShift
~~~

At this point, we should have the oc tool available for use.  Let's test this
out by printing the version of the oc command:

~~~shell
$ oc version
~~~

You should see the OpenShift version.

#### Download and Install OpenShift CLI on Mac

Download the the OpenShift CLI tool for [Mac]({{DOWNLOAD_CLIENT_MAC}})

Once the file has been downloaded, you will need to extract the contents as it
is a compressed archive. I would suggest saving this file to the following
directories:

~~~shell
~/openShift
~~~

Open up a terminal window and change to the directory where you downloaded the
file. Once you are in the directory, enter in the following command:

~~~shell
$ tar zxvf oc-macosx.tar.gz
~~~

The tar.gz file name needs to be replaced by the entire name that was downloaded in the previous step.

Now you can add the OpenShift CLI tools to your PATH.

~~~shell
$ export PATH=$PATH:~/openShift
~~~

At this point, we should have the oc tool available for use.  Let's test this
out by printing the version of the oc command:

~~~shell
$ oc version
~~~

You should see the OpenShift version.


#### Download Lab Projects

In order to get started, you need a few project skeletons to skip building those during 
the lab. 

Download the project skeletons to your local machine:

~~~shell
$ cd ~
$ curl -skL -o projects.zip {{LABS_DOWNLOAD_URL}}
~~~

> You can choose any directory, these instructions use `$HOME` as an example.

Unzip the `projects.zip` file in your home directory.

> You can use archiving utility with `zip` format support you have available on your machine.

* Windows: `Expand-Archive -Path .\projects.zip ; Move-Item .\projects\cloud-native-labs-master\* .`
* Linux: `unzip projects.zip && mv cloud-native-labs-master/* .`
* Mac: `tar xvfz projects.zip --strip-components 1`

After unzipping the projects, you should see these folders.

~~~
$ ls -l

-rwxr-xr-x  1 user  wheel  1718 Aug 14 14:50 README.md
drwxr-xr-x  6 user  wheel   204 Aug 14 14:50 catalog-spring-boot
drwxr-xr-x  6 user  wheel   204 Aug 14 14:50 gateway-vertx
drwxr-xr-x  6 user  wheel   204 Aug 14 14:50 inventory-wildfly-swarm
drwxr-xr-x  9 user  wheel   306 Aug 14 14:50 solutions
drwxr-xr-x  8 user  wheel   272 Aug 14 14:50 web-nodejs
~~~


## Explore OpenShift with OpenShift CLI

In order to login, we will use the `oc` command and then specify the server that we
want to authenticate to.

Issue the following command and replace `{{OPENSHIFT_CONSOLE_URL}}` 
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

> On some versions of Microsoft Windows, you may get an error that the
> server has an invalid x.509 certificate. If you receive this error, enter in
> the following command and replace `{{OPENSHIFT_CONSOLE_URL}}` with your 
> OpenShift Web Console url: 
>     
>     $ oc login {{OPENSHIFT_CONSOLE_URL}} --insecure-skip-tls-verify=true
>     

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
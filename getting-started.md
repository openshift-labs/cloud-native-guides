## Getting Started with OpenShift

In this lab you will get familiar with the OpenShift CLI and OpenShift Web Console 
and get ready for the Cloud Native Roadshow labs.

#### Prerequisites

In order to perform the labs, you will need the following installed in your workstation:

* Java Development Kit 8
* Apache Maven 3.3.x 
* Git client
* A Text Editor (e.g. Atom, Visual Studio Code) or an IDE (JBoss Developer Studio, Eclipse, IntelliJ)

{% if MINISHIFT == true %}

#### Red Hat Container Development Kit (CDK)

[Red Hat Container Development Kit](https://developers.redhat.com/products/cdk/overview)
provides a pre-built Container Development 
Environment based on Red Hat Enterprise Linux to help you develop container-based 
applications quickly on OpenShift. 

CDK configures a pre-built, single-node OpenShift cluster locally, so you can try 
the latest version of OpenShift Container Platform. 

If you haven't already installed CDK, follow these instructions to install it now: [Installing Container Development Kit](https://access.redhat.com/documentation/en-us/red_hat_container_development_kit/3.1/html/getting_started_guide/getting_started_with_container_development_kit#installing-minishift)

{% else %}

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
$ tar zxvf oc.tar.gz
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
$ tar zxvf oc.tar.gz
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

{% endif %}

#### Explore OpenShift with OpenShift CLI

In order to login, we will use the `oc` command and then specify the server that we
want to authenticate to.

{% if MINISHIFT == true %}

When CDK starts, it prints the OpenShift Web Console in the logs. Alternatively, 
you can use the `minishift console --url` to find it out. 

Login to OpenShift.

~~~shell
$ oc login {{OPENSHIFT_CONSOLE_URL}}
~~~

{% else %}

Issue the following command and replace `{{OPENSHIFT_CONSOLE_URL}}` 
with your OpenShift Web Console url:

~~~shell
$ oc login {{OPENSHIFT_CONSOLE_URL}}
~~~

{% endif %}

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

~~~shell
$ ls -l

-rwxr-xr-x  1 user  wheel  1718 Aug 14 14:50 README.md
drwxr-xr-x  6 user  wheel   204 Aug 14 14:50 catalog-spring-boot
drwxr-xr-x  6 user  wheel   204 Aug 14 14:50 gateway-vertx
drwxr-xr-x  6 user  wheel   204 Aug 14 14:50 inventory-wildfly-swarm
drwxr-xr-x  9 user  wheel   306 Aug 14 14:50 solutions
drwxr-xr-x  8 user  wheel   272 Aug 14 14:50 web-nodejs
~~~

Now you are ready to get started with the labs!

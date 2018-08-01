##  Automating Deployments Using Pipelines

In this lab you will learn about deployment pipelines and you will create a pipeline to 
automate build and deployment of the Inventory service.

#### Continuous Delivery
So far you have been building and deploying each service manually to OpenShift. Although 
it's convenient for local development, it's an error-prone way of delivering software if 
extended to test and production environments.

Continuous Delivery (CD) refers to a set of practices with the intention of automating 
various aspects of delivery software. One of these practices is called delivery pipeline 
which is an automated process to define the steps a change in code or configuration has 
to go through in order to reach upper environments and eventually to production. 

OpenShift simplifies building CI/CD Pipelines by integrating
the popular [Jenkins pipelines](https://jenkins.io/doc/book/pipeline/overview/) into
the platform and enables defining truly complex workflows directly from within OpenShift.

The first step for any deployment pipeline is to store all code and configurations in 
a source code repository.

#### Create a Git Repository for Inventory

You can use any Git server (e.g. GitHub, BitBucket, etc) for this lab but we have prepared a 
Gogs git server which you can access here: 

{{ GIT_URL }}

Click on **Register** to register a new user with the following details and then click on 
**Create New Account**: 

* Username: _same as your OpenShift user_
* Email: *your email*  (Don't worry! Gogs won't send you any emails)
* Password: `openshift`

![Sign Up Gogs]({% image_path cd-gogs-signup.png %}){:width="900px"}

You will be redirected to the sign in page. Sign in using the above username and password.

Click on the plus icon on the top navigation bar and then on **New Repository**.

![Create New Repository]({% image_path cd-gogs-plus-icon.png %}){:width="900px"}

Give `inventory-wildfly-swarm` as **Repository Name** and click on **Create Repository** 
button, leaving the rest with default values.

![Create New Repository]({% image_path cd-gogs-new-repo.png %}){:width="700px"}

The Git repository is created now. 

Click on the copy-to-clipboard icon to near the 
HTTP Git url to copy it to the clipboard which you will need in a few minutes.

![Empty Repository]({% image_path cd-gogs-empty-repo.png %}){:width="900px"}

#### Push Inventory Code to the Git Repository

Now that you have a Git repository for the Inventory service, you should push the 
source code into this Git repository.

Go the `inventory-wildfly-swarm` folder, initialize it as a Git working copy and add 
the GitHub repository as the remote repository for your working copy. 

> Replace `GIT-REPO-URL` with the Git repository url copied in the previous steps

~~~shell
$ cd labs/inventory-wildfly-swarm
$ git init
$ git remote add origin GIT-REPO-URL
~~~


Before you commit the source code to the Git repository, configure your name and 
email so that the commit owner can be seen on the repository. If you want, you can 
replace the name and the email with your own in the following commands:

~~~shell
git config --global user.name "Developer"
git config --global user.email "developer@me.com"
~~~

Commit and push the existing code to the GitHub repository.

~~~shell
$ git add . --all
$ git commit -m "initial add"
$ git push -u origin master
~~~

Enter your Git repository username and password if you get asked to enter your credentials. Go 
to your `inventory-wildfly-swarm` repository web interface and refresh the page. You should 
see the project files in the repository.

![Inventory Repository]({% image_path cd-gogs-inventory-repo.png %}){:width="900px"}

#### Define the Deployment Pipeline

OpenShift has built-in support for CI/CD pipelines by allowing developers to define 
a [Jenkins pipeline](https://jenkins.io/solutions/pipeline/) for execution by a Jenkins 
automation engine, which is automatically provisioned on-demand by OpenShift when needed.

The build can get started, monitored, and managed by OpenShift in 
the same way as any other build types e.g. S2I. Pipeline workflows are defined in 
a Jenkinsfile, either embedded directly in the build configuration, or supplied in 
a Git repository and referenced by the build configuration. 

Jenkinsfile is a text file that contains the definition of a Jenkins Pipeline 
and is created using a [scripted or declarative syntax](https://jenkins.io/doc/book/pipeline/syntax/).

In the project explorer in Eclipse Che, right-click on `inventory-wildfly-swarm` project and then 
on **New > File** and name it `Jenkinsfile`.

Copy the following pipeline definition into `Jenkinsfile`.

~~~shell
pipeline {
  agent {
      label 'maven'
  }
  stages {
    stage('Build JAR') {
      steps {
        sh "mvn package"
        stash name:"jar", includes:"target/inventory-1.0-SNAPSHOT-swarm.jar"
      }
    }
    stage('Build Image') {
      steps {
        unstash name:"jar"
        script {
          openshift.withCluster() {
            openshift.startBuild("inventory-s2i", "--from-file=target/inventory-1.0-SNAPSHOT-swarm.jar", "--wait")
          }
        }
      }
    }
    stage('Deploy') {
      steps {
        script {
          openshift.withCluster() {
            def dc = openshift.selector("dc", "inventory")
            dc.rollout().latest()
            dc.rollout().status()
          }
        }
      }
    }
  }
}
~~~

This pipeline has three stages:

* *Build JAR*: to build and test the jar file using Maven
* *Build Image*: to build a container image from the Inventory JAR archive using OpenShift S2I
* *Deploy*: to deploy the Inventory container image in the current project

Note that the pipeline definition is fully integrated with OpenShift and you can 
perform operations like image build, image deploy, etc directly from within the `Jenkinsfile`.

When building deployment pipelines, it's important to treat your [infrastructure and everything else that needs to be configured (including the pipeline definition) as code](https://martinfowler.com/bliki/InfrastructureAsCode.html) 
and store them in a source repository for version control. 

Commit and push the `Jenkinsfile` to the Git repository.

~~~shell
$ git add Jenkinsfile
$ git commit -m "pipeline added"
$ git push origin master
~~~

The pipeline definition is ready and now you can create a deployment pipeline using 
this `Jenkinsfile`.

#### Create an OpenShift Pipeline

Like mentioned, [OpenShift Pipelines]({{OPENSHIFT_DOCS_BASE}}/architecture/core_concepts/builds_and_image_streams.html#pipeline-build) enable creating deployment pipelines using the widely popular `Jenkinsfile` format.

OpenShift automates deployments using [deployment triggers]({{OPENSHIFT_DOCS_BASE}}/dev_guide/deployments/basic_deployment_operations.html#triggers) that react to changes to the container image or configuration. Since you want to control the deployments instead 
from the pipeline, you should remove the Inventory deploy triggers so that building a new 
Inventory container image wouldn't automatically result in a new deployment. That would 
allow the pipeline to decide when a deployment should occur.

Remove the Inventory deployment triggers:

~~~shell
$ oc set triggers dc/inventory --manual
~~~

Deploy a Jenkins server using the provided template and container image that 
comes out-of-the-box with OpenShift:

```
oc new-app jenkins-ephemeral
```

After Jenkins is deployed and is running (verify in web console), then create a 
deployment pipeline by running the following command within the `inventory-widlfly-swarm` folder:

~~~shell
$ oc new-app . --name=inventory-pipeline --strategy=pipeline
~~~

The above command creates a new build config of type pipeline which is automatically 
configured to fetch the `Jenkinsfile` from the Git repository of the current folder 
(`inventory-wildfly-swarm` Git repository) and execute it on Jenkins.

Go OpenShift Web Console inside the **{{COOLSTORE_PROJECT}}** project and from the left sidebar 
click on **Builds >> Pipelines**

![OpenShift Pipeline]({% image_path cd-pipeline-inprogress.png %}){:width="900px"}

Pipeline syntax allows creating complex deployment scenarios with the possibility of defining 
checkpoint for manual interaction and approval process using 
[the large set of steps and plugins that Jenkins provide](https://jenkins.io/doc/pipeline/steps/) in 
order to adapt the pipeline to the process used in your team. You can see a few examples of 
advanced pipelines in the 
[OpenShift GitHub Repository](https://github.com/openshift/origin/tree/master/examples/jenkins/pipeline).

In order to update the deployment pipeline, all you need to do is to update the `Jenkinsfile` 
in the `inventory-wildfly-swarm` Git repository. OpenShift pipeline automatically executes the 
updated pipeline next time it runs.

#### Run the Pipeline on Every Code Change

Manually triggering the deployment pipeline to run is useful but the real goes is to be able 
to build and deploy every change in code or configuration at least to lower environments 
(e.g. dev and test) and ideally all the way to production with some manual approvals in-place.

In order to automate triggering the pipeline, you can define a webhook on your Git repository 
to notify OpenShift on every commit that is made to the Git repository and trigger a pipeline 
execution.

You can get see the webhook links in the OpenShift Web Console by going to **Build >> Pipelines**, clicking 
on the pipeline and going to the **Configurations** tab.

Copy the Generic webhook url which you will need in the next steps.

Go to Gogs and your **inventory-wildfly-swarm** Git repository, then click on **Settings**.

![Repository Settings]({% image_path cd-gogs-settings-link.png %}){:width="900px"}

On the left menu, click on **Webhooks** and then on **Add Webhook** button and then **Gogs**. 

Create a webhook with the following details:

* **Payload URL**: paste the Generic webhook url you copied from the `inventory-pipeline`
* **Content type**: `application/json`

Click on **Add Webhook**. 

![Repository Webhook]({% image_path cd-gogs-webhook-add.png %}){:width="660px"}

All done. You can click on the newly defined webhook to see the list of *Recent Delivery*. 
Clicking on the **Test Delivery** button allows you to manually trigger the webhook for 
testing purposes. Click on it and verify that the `inventory-pipeline` start running 
immediately.

Well done! You are ready for the next lab.

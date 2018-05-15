Cloud-Native Roadshow [![Build Status](https://travis-ci.org/openshift-labs/cloud-native-guides.svg?branch=ocp-3.9)](https://travis-ci.org/openshift-labs/cloud-native-guides)
===
This one day hands-on cloud-native workshops provides developers and introduction to cloud-natives applications and gives them an experience of building cloud-native applications using OpenShift, Spring Boot, WildFly Swarm, Vert.xt and more.

Agenda
===
* Introduction to Cloud-Native apps
* Building services with Spring Boot
* Building Java EE services with WildFly Swarm
* Building Reactive Services with Vert.x
* Monitoring Application Health
* Fault Tolerance and Service Resilience
* Configuration Management 
* Continuous Delivery 
* Debugging Services


Install Workshop Infrastructure
===

An [APB](https://hub.docker.com/r/openshiftapb/cloudnative-workshop-apb) is provided for 
deploying the Cloud-Native Workshop infra (lab instructions, Nexus, Gogs, Eclipse Che, etc) in a project 
on an OpenShift cluster via the service catalog. [Read more in the docs](https://docs.openshift.com/container-platform/3.9/install_config/oab_broker_configuration.html#oab-config-registry-dockerhub) 
on how to add an APB from DockerHub to the service catalog.

Note that if you are using the _OpenShift Workshop_ in RHPDS, this APB is already available in your service catalog.

![](images/service-catalog.png?raw=true)

As an alternative, you can also run the APB directly in a pod on OpenShift to install the workshop infra:

```
oc login
oc new-project lab-infra
oc run apb --restart=Never --image="openshiftapb/cloudnative-workshop-apb:ocp-3.9" \
    -- provision -vvv -e namespace=$(oc project -q) -e openshift_token=$(oc whoami -t)
```

Lab Instructions on OpenShift
===

Note that if you have used the above workshop installer, the lab instructions are already deployed.

```
$ oc new-app osevg/workshopper:latest --name=guides \
    -e WORKSHOPS_URLS="https://raw.githubusercontent.com/openshift-labs/cloud-native-guides/ocp-3.9/_cloud-native-roadshow.yml"
$ oc expose svc/guides
```

Local Lab Instructions
===
```
$ docker run -p 8080:8080 \
              -v /path/to/clone/dir:/app-data \
              -e CONTENT_URL_PREFIX="file:///app-data" \
              -e WORKSHOPS_URLS="file:///app-data/_cloud-native-roadshow.yml" \
              osevg/workshopper:latest
```

Cloud-Native Workshop [![Build Status](https://travis-ci.org/openshift-labs/cloud-native-guides.svg?branch=ocp-3.11)](https://travis-ci.org/openshift-labs/cloud-native-guides)
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
on an OpenShift cluster via the service catalog. In order to add this APB to the OpenShift service catalog, log in 
as cluster admin and perform the following in the `openshift-ansible-service-broker` project :

1. Edit the `broker-config` configmap and add this snippet right after `registry:`:

  ```
    - name: dh
      type: dockerhub
      org: openshiftapb
      tag: ocp-3.11
      white_list: [.*-apb$]
  ```

2. Redeploy the `asb` deployment

You can [read more in the docs](https://docs.openshift.com/container-platform/3.11/install_config/oab_broker_configuration.html#oab-config-registry-dockerhub) 
on how to configure the service catalog.

Note that if you are using the _OpenShift Workshop_ in RHPDS, this APB is already available in your service catalog.

![](images/service-catalog.png?raw=true)

As an alternative, you can also run the APB directly in a pod on OpenShift to install the workshop infra:

```
oc login
oc new-project lab-infra
oc run apb --restart=Never --image="openshiftapb/cloudnative-workshop-apb:ocp-3.11" \
    -- provision -vvv -e namespace=$(oc project -q) -e openshift_token=$(oc whoami -t)
```

Or if you have Ansible installed locally, you can also run the Ansilbe playbooks directly on your machine:

```
oc login
oc new-project lab-infra

ansible-playbook -vvv playbooks/provision.yml \
       -e namespace=$(oc project -q) \
       -e openshift_token=$(oc whoami -t) \
       -e openshift_master_url=$(oc whoami --show-server)
``` 

Lab Instructions on OpenShift
===

Note that if you have used the above workshop installer, the lab instructions are already deployed.

```
$ oc new-app quay.io/osevg/workshopper:latest --name=guides \
    -p LOG_TO_STDOUT=true \
    -p WORKSHOPS_URLS="https://raw.githubusercontent.com/openshift-labs/cloud-native-guides/ocp-3.11/_cloud-native-workshop.yml"
$ oc expose svc/guides
```

Local Lab Instructions
===
```
$ docker run -it --rm -p 8080:8080 \
      -v $(pwd):/app-data \
      -e LOG_TO_STDOUT=true \
      -e CONTENT_URL_PREFIX="file:///app-data" \
      -e WORKSHOPS_URLS="file:///app-data/_cloud-native-workshop.yml" \
      quay.io/osevg/workshopper:latest
```

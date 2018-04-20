Cloud-Native Roadshow
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

Prepare OpenShift Cluster for Workshop
===
An [Ansible playbook](ansible/) is provided for preparing an OpenShift cluster.

Deploy on OpenShift
===
```
$ oc new-app osevg/workshopper:latest --name=guides \
    -e WORKSHOPS_URLS="https://raw.githubusercontent.com/openshift-labs/cloud-native-guides/ocp-3.9/_cloud-native-roadshow.yml"
$ oc expose svc/guides
```

Run Locally
===
```
$ docker run -p 8080:8080 \
              -v /path/to/clone/dir:/app-data \
              -e CONTENT_URL_PREFIX="file:///app-data" \
              -e WORKSHOPS_URLS="file:///app-data/_cloud-native-roadshow.yml" \
              osevg/workshopper:latest
```
Cloud-Native Roadshow
===
This one day hands-on cloud-native workshops provides developers and introduction to cloud-natives applications and gives them an experience of building cloud-native applications using OpenShift, Spring Boot, WildFly Swarm, Vert.xt and more.

Agenda
===
* Introduction to Cloud-Native apps
* Building services with Spring Boot
* Building Java EE services with WildFly Swarm
* Building Reactive Services with Vert.x
* Rolling updates and deployments (demo)
* Monitoring Application Health
* Fault Tolerance and Service Resilience
* Debugging Services
* Configuration Management 
* Service Discovery and Load Balancing
* Continuous Delivery 
* OpenShift.io (demo)


Deploy Guides on OpenShift
===
```
$ oc new-build . --name=guides
$ oc start-build guides --from-dir=.
$ oc new-app --name=guides --image-stream=guides
$ oc expose svc/guides
$ oc set probe dc/guides --readiness --liveness --get-url=http://:8080/ --failure-threshold=5 --initial-delay-seconds=15
```


TODO: Add more desc to the lab goals
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
* OpenShift.io (demo)


Deploy on OpenShift with Embedded Content
===
```
$ oc new-build . --name=guides
$ oc start-build guides --from-dir=.
$ oc new-app --name=guides --image-stream=guides
$ oc set probe dc/guides --readiness --liveness --get-url=http://:8080/ --failure-threshold=5 --initial-delay-seconds=15
$ oc expose svc/guides
```

Deploy on OpenShift Online with Embedded Content
===
```
$ oc new-project roadshow

$ docker login -u USERNAME -p $(oc whoami -t) https://registry.CLUSTER-ID.openshift.com
$ docker build -t registry.CLUSTER-ID.openshift.com/roadshow/guides .
$ docker push registry.CLUSTER-ID.openshift.com/roadshow/guides

$ oc new-app --name=guides --image-stream=guides
$ oc set probe dc/guides --readiness --liveness --get-url=http://:8080/ --failure-threshold=5 --initial-delay-seconds=15
$ oc expose svc/guides
```

Deploy on OpenShift with GitHub Content
===
```
$ oc process -f openshift/template.yml | oc create -f -
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
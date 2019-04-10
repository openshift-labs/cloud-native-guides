~~~shell
$ oc new-app https://github.com/mcouliba/cloud-native-labs.git#ocp-3.11 \
    --strategy=docker \
    --context-dir=catalog-go \
    --name=catalog-v2 \
    --labels app=catalog,version=2.0
$ selector svc/catalog = app=catalog #USELESS??
$ oc patch dc/catalog-v2 --patch '{"spec": {"template": {"metadata": {"annotations": {"sidecar.istio.io/inject": "true"}}}}}'

$ cat << EOF | oc create -f -
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: catalog
spec:
  host: catalog
  subsets:
  - labels:
      version: "1.0-SNAPSHOT"
    name: "version-springboot"
  - labels:
      version: "2.0"
    name: "version-go"
EOF
$ cat << EOF | oc replace -f -
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
    - catalog
  http:
  - route:
    - destination:
        host: catalog
        subset: "version-springboot"
      weight: 100
    - destination:
        host: catalog
        subset: "version-go"
      weight: 0
EOF
~~~
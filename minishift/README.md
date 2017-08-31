# Cloud Native Labs for OpenShift Roadshow Add-on for CDK and Minishift
An addon to install [OpenShift's Roadshow Cloud Native Labs](https://github.com/openshift-roadshow/cloud-native-labs) and [guide](https://github.com/openshift-roadshow/cloud-native-guides).

Verify you have installed these addons, by following the [General Readme](https://github.com/minishift/minishift-addons#download-and-use-community-add-ons).

## Deploy 
To deploy:

```
$ minishift addon apply cloud-native-labs
```

## Use
Find the guide at the following URL:

```
$ minishift openshift service guides -n cloud-native-labs
```

## Delete
To delete the guide, just do:

```
$ oc delete project/cloud-native-labs --as=system:admin
```
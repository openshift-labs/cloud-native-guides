Ansible Playbook For Workshop Setup
=========

The provided playbook automates preparing an OpenShift cluster for the Cloud-Native Labs 
by deploying required services (Gogs, Nexus, etc) which are used during the labs.

Playbook Variables
------------

| Variable              | Default Value | Description   |
|-----------------------|---------------|---------------|
|`lab_infra_project`    | `lab-infra`   | Project name to deploy Git server and lab guides  |
|`user_gogs_admin`      | `gogs`        | Admin user to be created in Gogs |
|`user_gogs_user`       | `developer`   | A sample user to be created in Gogs |
|`user_gogs_password`   | `openshift`   | Gogs password to configure for admin and sample user |
|`project_suffix`       | `-XX`         | Project suffix for project names to be created e.g. `coolstore-{PROJECT_SUFFIX}` |
|`install_eclipse_che`  | `false`         | Install Eclipse Che multi-user. Requires to be logged in as a cluster admin user |
|`clean_init`           | `false`       | Clean the environment and remove projects before init |


How To Run 
------------

Log in to OpenShift and then use the provided playbook:

```
oc login -u admin
ansible-galaxy install -r requirements.yml
ansible-playbook init.yml -e install_eclipse_che=true
```

How To Run without Ansible
------------ 

```
oc login -u admin
docker run --rm siamaksade/workshop-cloudnative-init:ocp-3.9 init.yml \
    -e openshift_master=https://master.sto1.openshiftworkshop.com \
    -e oc_token=$(oc whoami -t) \
    -e install_eclipse_che=true \
    -e clean_init=true
```
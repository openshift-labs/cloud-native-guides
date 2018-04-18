Ansible Playbook For Workshop Setup
=========

The provided playbook automates preparing an OpenShift cluster for the Cloud-Native Labs 
by deploying required services (Gogs, Nexus, etc) which are used during the labs.

Playbook Variables
------------

| Variable              | Default Value | Description   |
|-----------------------|---------------|---------------|
|`lab_infra_project`    | `lab-infra`   | Project name to deploy Git server and lab guides  |
|`user_gogs_admin`      | `gogs`        | Admin username to create in Gogs |
|`user_gogs_test`       | `test`        | Test username to create in Gogs |
|`user_gogs_password`   | `openshift`   | Gogs password to configure for admin and test users |
|`project_suffix`       | `-XX`         | Project suffix for project names to be created e.g. `coolstore-{PROJECT_SUFFIX}` |
|`clean_init`           | `false`       | Clean the environment and remove projects before init |


How To Run
------------

Log in to OpenShift and then use the provided playbook:

```
ansible-galaxy install -r requirements.yml
ansible-playbook init.yml 
```
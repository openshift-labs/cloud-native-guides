##  Managing Application Configuration

In this lab you will learn how to manage application configuration and how to provide environment 
specific configuration to the services.

#### Application Configuration

Applications require configuration in order to tweak the application behavior 
or adapt it to a certain environment without the need to write code and repackage 
the application for every change. These configurations are sometimes specific to 
the application itself such as the number of products to be displayed on a product 
page and some other times they are dependent on the environment they are deployed in 
such as the database coordinates for the application.

The most common way to provide configurations to applications is using environment 
variables and external configuration files such as properties, JSON or YAML files.

configuration files and command line arguments. These configuration artifacts
should be externalized from the application and the docker image content in
order to keep the image portable across environments.

OpenShift provides a mechanism called [ConfigMaps]({{OPENSHIFT_DOCS_BASE}}/dev_guide/configmaps.html) 
in order to externalize configurations 
from the applications deployed within containers and provide them to the containers 
in a unified way as files and environment variables. OpenShift also offers a way to 
provide sensitive configuration data such as certificates, credentials, etc to the 
application containers in a secure and encrypted mechanism called Secrets.

This allows developers to build the container image for their application only once, 
and reuse that image to deploy the application across various environments with 
different configurations that are provided to the application at runtime.

#### Create PostgreSQL Databases for Inventory and Catalog

So far Catalog and Inventory services have been using an in-memory H2 database. Although H2 
is a convenient database to run locally on your laptop, it's in no way appropriate for production or 
even integration tests. Since it's strongly recommended to use the same technology stack (operating 
system, JVM, middleware, database, etc) that is used in production across all environments, you 
should modify Inventory and Catalog services to use PostgreSQL instead of the H2 in-memory database.

Fortunately, OpenShfit supports stateful applications such as databases which require access to 
a persistent storage that survives the container itself. You can deploy databases on OpenShift and 
regardless of what happens to the container itself, the data is safe and can be used by the next 
database container.

Let's create a [PostgreSQL database]({{OPENSHIFT_DOCS_BASE}}/using_images/db_images/postgresql.html) 
for the Inventory service using the PostgreSQL template that is provided out-of-the-box:

> [OpenShift Templates]({{OPENSHIFT_DOCS_BASE}}/dev_guide/templates.html) uses YAML/JSON to compose 
> multiple containers and their configurations as a list of objects to be created and deployed at once hence 
> making it simple to re-create complex deployments by just deploying a single template. Templates can 
> be parameterized to get input for fields like service names and generate values for fields like passwords.

~~~shell
$ oc new-app postgresql-persistent \
    --param=DATABASE_SERVICE_NAME=inventory-postgresql \
    --param=POSTGRESQL_DATABASE=inventory \
    --param=POSTGRESQL_USER=inventory \
    --param=POSTGRESQL_PASSWORD=inventory \
    --labels=app=inventory
~~~

> The `--param` parameter provides a value for the template parameters. The recommended approach is 
> not to provide any value for username and password and allow the template to generate a random value for 
> you due to security reasons. In this lab in order to reduce typos, a fixed value is provided for username and 
> password.

Deploy another PostgreSQL database for the Catalog service:

~~~shell
$ oc new-app postgresql-persistent \
    --param=DATABASE_SERVICE_NAME=catalog-postgresql \
    --param=POSTGRESQL_DATABASE=catalog \
    --param=POSTGRESQL_USER=catalog \
    --param=POSTGRESQL_PASSWORD=catalog \
    --labels=app=catalog
~~~

Now you can move on to configure the Inventory and Catalog service to use these PostgreSQL databases.

#### Externalize WildFly Swarm (Inventory) Configuration

WildFly Swarm supports multiple mechanisms for externalizing configurations such as environment variables, 
Maven properties, command-line arguments and more. The recommend approach for the long-term for externalizing 
configuration is however using a [YAML file](https://reference.wildfly-swarm.io/configuration.html#_using_yaml) 
which you have already packaged within the Inventory Maven project.

The YAML file can be packaged within the application JAR file and be overladed 
[using command-line or system properties](https://wildfly-swarm.gitbooks.io/wildfly-swarm-users-guide/configuration/command_line.html) which you will do in this lab.

> Check out `inventory-wildfly-swarm/src/main/resources/project-defaults.yml` which contains the default configuration.

Create a YAML file with the PostgreSQL database credentials. Note that you can give an arbitrary 
name to this configuration (e.g. `prod`) in order to tell WildFly Swarm which one to use:

~~~shell
$ cat <<EOF > ./project-defaults.yml
swarm:
  datasources:
    data-sources:
      InventoryDS:
        driver-name: postgresql
        connection-url: jdbc:postgresql://inventory-postgresql:5432/inventory
        user-name: inventory
        password: inventory
EOF
~~~

> The hostname defined for the PostgreSQL connection-url corresponds to the PostgreSQL 
> service name published on OpenShift. This name will be resolved by the internal DNS server 
> exposed by OpenShift and accessible to containers running on OpenShift.

And then create a config map that you will use to overlay on the default `project-defaults.yml` which is 
packaged in the Inventory JAR archive:

~~~shell
$ oc create configmap inventory --from-file=./project-defaults.yml
~~~

> If you don't like bash commands, Go to the **{{COOLSTORE_PROJECT}}** 
> project in OpenShift Web Console and then on the left sidebar, **Resources >> Config Maps**. Click 
> on **Create Config Map** button to create a config map with the following info:
> 
> * Name: `inventory`
> * Key: `project-defaults.yml`
> * Value: *copy-paste the content of the above project-defaults.yml excluding the first and last lines (the lines that contain EOF)*

Config maps hold key-value pairs and in the above command an `inventory` config map 
is created with `project-defaults.yml` as the key and the content of the `project-defaults.yml` as the 
value. Whenever a config map is injected into a container, it would appear as a file with the same 
name as the key, at specified path on the filesystem.

> You can see the content of the config map in the OpenShift Web Console or by 
> using `oc describe cm inventory` command.

Modify the Inventory deployment config so that it injects the YAML configuration you just created as 
a config map into the Inventory container:

~~~shell
$ oc volume dc/inventory --add --configmap-name=inventory --mount-path=/app/config
~~~

The above command mounts the content of the `inventory` config map as a file inside the Inventory container 
at `/app/config/project-defaults.yaml`

The last step is the [aforementioned system properties](https://wildfly-swarm.gitbooks.io/wildfly-swarm-users-guide/configuration/command_line.html) on the Inventory container to overlay the WildFly Swarm configuration, using the `JAVA_ARGS` environment variable. 

> The Java runtime on OpenShift can be configured using 
> [a set of environment variables](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/reference#configuration_environment_variables) 
> to tune the JVM without the need to rebuild a new Java runtime container image every time a new option is needed.

~~~shell
$ oc set env dc/inventory JAVA_ARGS="-s /app/config/project-defaults.yml"
~~~


The Inventory pod gets restarted automatically due to the configuration changes. Wait till it's ready, 
and then verify that the config map is in fact injected into the container by running 
a shell command inside the Inventory container:

~~~shell
$ oc rsh dc/inventory cat /app/config/project-defaults.yml
~~~

Also verify that the PostgreSQL database is actually used by the Inventory service. Check the 
Inventory pod logs:

~~~shell
oc logs dc/inventory | grep hibernate.dialect

2017-08-10 16:55:44,657 INFO  [org.hibernate.dialect.Dialect] (ServerService Thread Pool -- 15) HHH000400: Using dialect: org.hibernate.dialect.PostgreSQL94Dialect
~~~

You can also connect to Inventory PostgreSQL database and check if the seed data is 
loaded into the database.

~~~shell
$ oc rsh dc/inventory-postgresql
~~~

Once connected to the PostgreSQL container, run the following:

> Run this command inside the Inventory PostgreSQL container, after opening a remote shell to it.

~~~shell
$ psql -U inventory -c "select * from inventory"

 itemid | quantity
------------------
 329299 |       35
 329199 |       12
 165613 |       45
 165614 |       87
 165954 |       43
 444434 |       32
 444435 |       53
 444436 |       42
(8 rows)

$ exit
~~~

You have now created a config map that holds the configuration content for Inventory and can be updated 
at anytime for example when promoting the container image between environments without needing to 
modify the Inventory container image itself. 

#### Externalize Spring Boot (Catalog) Configuration

You should be quite familiar with config maps by now. Spring Boot application configuration is provided 
via a properties file called `application.properties` and can be 
[overriden and overlayed via multiple mechanisms](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html). 

> Check out the default Spring Boot configuration in Catalog Maven project `catalog-spring-boot/src/main/resources/application.properties`.

In this lab, you will configure the Catalog service which is based on Spring Boot to override the default 
configuration using an alternative `application.properties` backed by a config map.

Create a config map with the the Spring Boot configuration content using the PostgreSQL database 
credentials:

~~~shell
$ cat <<EOF > ./application.properties
spring.datasource.url=jdbc:postgresql://catalog-postgresql:5432/catalog
spring.datasource.username=catalog
spring.datasource.password=catalog
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.hibernate.ddl-auto=create
EOF
~~~

> The hostname defined for the PostgreSQL connection-url corresponds to the PostgreSQL 
service name published on OpenShift. This name will be resolved by the internal DNS server 
exposed by OpenShift and accessible to containers running on OpenShift.

~~~shell
$ oc create configmap catalog --from-file=./application.properties
~~~

> You can use the OpenShift Web Console to create config maps by clicking on **Resources >> Config Maps** 
> on the left sidebar inside the your project. Click on **Create Config Map** button to create a config map 
> with the following info:
> 
> * Name: `catalog`
> * Key: `application.properties`
> * Value: *copy-paste the content of the above application.properties excluding the first and last lines (the lines that contain EOF)*

The [Spring Cloud Kubernetes](https://github.com/spring-cloud-incubator/spring-cloud-kubernetes) plug-in implements 
the integration between Kubernetes and Spring Boot and is already added as a dependency to the Catalog Maven 
project. Using this dependency, Spring Boot would search for a config map (by default with the same name as 
the application) to use as the source of application configurations during application bootstrapping and 
if enabled, triggers hot reloading of beans or Spring context when changes are detected on the config map.

Although Spring Cloud Kubernetes tries to discover config maps, due to security reasons containers 
by default are not allowed to snoop around OpenShift clusters and discover objects. Security comes first, 
and discovery is a privilege that needs to be granted to containers in each project. 

Since you do want Spring Boot to discover the config maps inside the `{{COOLSTORE_PROJECT}}` project, you 
need to grant permission to the Spring Boot service account to access the OpenShift REST API and find the 
config maps. However you have done this already in previous labs and no need to grant permission again. 

> For the record, you can grant permission to the default service account in your project using this 
command: 
> 
>     $ oc policy add-role-to-user view -n {{COOLSTORE_PROJECT}} -z default

Delete the Catalog container to make it start again and look for the config maps:

~~~shell
$ oc delete pod -l app=catalog
~~~

When the Catalog container is ready, verify that the PostgreSQL database is being 
used. Check the Catalog pod logs:

~~~shell
$ oc logs dc/catalog | grep hibernate.dialect

2017-08-10 21:07:51.670  INFO 1 --- [           main] org.hibernate.dialect.Dialect            : HHH000400: Using dialect: org.hibernate.dialect.PostgreSQL94Dialect
~~~

You can also connect to the Catalog PostgreSQL database and verify that the seed data is loaded:

~~~shell
$ oc rsh dc/catalog-postgresql
~~~

Once connected to the PostgreSQL container, run the following:

> Run this command inside the Catalog PostgreSQL container, after opening a remote shell to it.

~~~shell
$ psql -U catalog -c "select item_id, name, price from product"

 item_id |            name             | price
----------------------------------------------
 329299  | Red Fedora                  | 34.99
 329199  | Forge Laptop Sticker        |   8.5
 165613  | Solid Performance Polo      |  17.8
 165614  | Ogio Caliber Polo           | 28.75
 165954  | 16 oz. Vortex Tumbler       |     6
 444434  | Pebble Smart Watch          |    24
 444435  | Oculus Rift                 |   106
 444436  | Lytro Camera                |  44.3
(8 rows)

$ exit
~~~

#### Sensitive Configuration Data

Config map is a superb mechanism for externalizing application configuration while keeping 
containers independent of in which environment or on what container platform they are running. 
Nevertheless, due to their clear-text nature, they are not suitable for sensitive data like 
database credentials, SSH certificates, etc. In the current lab, we used config maps for database 
credentials to simplify the steps however for production environments, you should opt for a more 
secure way to handle sensitive data.

Fortunately, OpenShift already provides a secure mechanism for handling sensitive data which is 
called [Secrets]({{OPENSHIFT_DOCS_BASE}}/dev_guide/secrets.html). Secret objects act and are used 
similar to config maps however with the difference that they are encrypted as they travel over the wire 
and also at rest when kept on a persistent disk. Like config maps, secrets can be injected into 
containers as environment variables or files on the filesystem using a temporary file-storage 
facility (tmpfs).

You won't create any secrets in this lab however you have already created 2 secrets when you created 
the PostgreSQL databases for Inventory and Catalog services. The PostgreSQL template by default stores 
the database credentials in a secret in the project it's being created:

~~~shell
$ oc describe secret catalog-postgresql

Name:            catalog-postgresql
Namespace:       coolstore
Labels:          app=catalog
                 template=postgresql-persistent-template
Annotations:     openshift.io/generated-by=OpenShiftNewApp
                 template.openshift.io/expose-database_name={.data['database-name']}
                 template.openshift.io/expose-password={.data['database-password']}
                 template.openshift.io/expose-username={.data['database-user']}

Type:     Opaque

Data
====
database-name:        7 bytes
database-password:    7 bytes
database-user:        7 bytes
~~~

This secret has two encrypted properties defined as `database-user` and `database-password` which hold 
the PostgreSQL username and password values. These values are injected in the PostgreSQL container as 
environment variables and used to initialize the database.

Go to the **{{COOLSTORE_PROJECT}}** in the OpenShift Web Console and click on the `catalog-postgresql` 
deployment (blue text under the title **Deployment**) and then on the **Environment**. Notice the values 
from the secret are defined as env vars on the deployment:

![Secrets as Env Vars]({% image_path config-psql-secret.png %}){:width="900px"}

That's all for this lab! You are ready to move on to the next lab.

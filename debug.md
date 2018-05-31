## Debugging Applications

In this lab you will debug the coolstore application using Java remote debugging and 
look into line-by-line code execution as the code runs inside a container on OpenShift.

#### Investigate The Bug

CoolStore application seem to have a bug that causes the inventory status for one of the 
products not be displayed in the web interface. 

![Inventory Status Bug]({% image_path debug-coolstore-bug.png %}){:width="800px"}

This is not an expected behavior! In previous labs, you added a circuit breaker to 
protect the coolstore application from failures and in case the Inventory API is not 
available, to skip it and show the products without the inventory status. However, right 
now the inventory status is available for all products but one which is not how we 
expect to see the products.

Since the product list is provides by the API Gateway, take a look into the API Gateway 
logs to see if there are any errors:

~~~shell
$ oc logs dc/gateway | grep -i error

...
WARNING: Inventory error for 444436: status code 204
SEVERE: Inventory error for 444436: null
...
~~~

Oh! Something seems to be wrong with the response the API Gateway has received from the 
Inventory API for the product id `444436`. 

Look into the Inventory pod logs to investigate further and see if you can find more  
information about this bug:


~~~shell
$ oc logs dc/inventory | grep ERROR
~~~

There doesn't seem to be anything relevant to the `invalid response` error that the 
API Gateway received either! 

Invoke the Inventory API using `curl` for the suspect product id to see what actually 
happens when API Gateway makes this call:

> You can find out the Inventory route url using `oc get route inventory`. Replace 
> `{{INVENTORY_ROUTE_HOST}}` with the Inventory route url from your project.

~~~shell
$ curl http://{{INVENTORY_ROUTE_HOST}}/api/inventory/444436
~~~

> You can use `curl -v` to see all the headers sent and received. You would received 
> a `HTTP/1.1 204 No Content` response for the above request.

No response came back and that seems to be the reason the inventory status is not displayed 
on the web interface.

Let's debug the Inventory service to get to the bottom of this!

#### Enable Remote Debugging 

Remote debugging is a useful debugging technique for application development which allows 
looking into the code that is being executed somewhere else on a different machine and 
execute the code line-by-line to help investigate bugs and issues. Remote debugging is 
part of  Java SE standard debugging architecture which you can learn more about it in [Java SE docs](https://docs.oracle.com/javase/8/docs/technotes/guides/jpda/architecture.html).


The Java image on OpenShift has built-in support for remote debugging and it can be enabled 
by setting the `JAVA_DEBUG=true` environment variables on the deployment config for the pod 
that you want to remotely debug.

An easier approach would be to use the fabric8 maven plugin to enable remote debugging on 
the Inventory pod. It also forwards the default remote debugging port, 5005, from the 
Inventory pod to your workstation so simplify connectivity.

Enable remote debugging on Inventory:

~~~shell
$ cd inventory-wildfly-swarm
$ mvn fabric8:debug
~~~~

> The default port for remoting debugging is `5005` but you can change the default port 
> via environment variables. Read more in the [Java S2I Image docs](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/reference#configuration_environment_variables).

You are all set now to start debugging using the tools of you choice. 

Do not wait for the command to return! The fabric8 maven plugin keeps the forwarded 
port open so that you can start debugging remotely.

Remote debugging can be done using the prevalently available
Java Debugger command line or any modern IDE like JBoss 
Developer Studio (Eclipse) and IntelliJ IDEA.

{% if REMOTE_DEBUGGER_JDB == true %}

#### Debug Remotely Using JDB (Java Debugger)

The [Java Debugger (JDB)](http://docs.oracle.com/javase/8/docs/technotes/tools/windows/jdb.html) 
is a simple command-line debugger for Java. The `jdb` command is included by default in 
Java SE and provides inspection and debugging of a local or remote JVM. Although JDB is not 
the most convenient way to debug Java code, it is a handy tool since it can be run on any environment 
that Java SE is available.

In a new terminal window, go to the `inventory-wildfly-swarm` project folder 
and start JDB by pointing at the folder 
containing the Java source code for the application under debug:

~~~shell
$ jdb -attach localhost:5005 -sourcepath :src/main/java/

Set uncaught java.lang.Throwable
Set deferred uncaught java.lang.Throwable
Initializing jdb ...
>
~~~

Now that you are connected to the JVM running inside the Inventory pod on OpenShift, add 
a breakpoint to pause the code execution when it reaches the Java method handling the 
REST API `/api/inventory`. Review `com/redhat/cloudnative/inventory/InventoryResource.java` and note that the 
`getAvailability()` is the method where you should add the breakpoint.

Add a breakpoint.

~~~shell
> stop in com.redhat.cloudnative.inventory.InventoryResource.getAvailability
~~~

Use `curl` to invoke the Inventory API with the suspect product id in a 
a new terminal window in order to pause the code execution at the defined breakpoint.

> You can find out the Inventory route url using `oc get routes`. Replace 
> `{{INVENTORY_ROUTE_HOST}}` with the Inventory route url from your project.

~~~shell
$ curl -v http://{{INVENTORY_ROUTE_HOST}}/api/inventory/444436
~~~

The code execution pauses at the `getAvailability()` method. You can verify it 
using the `list` command to see the source code. The arrow shows which line is 
to execute next:

~~~shell
> list
~~~

You'll see an output similar to this.

~~~shell
default task-3[1] list
21        @GET
22        @Path("/api/inventory/{itemId}")
23        @Produces(MediaType.APPLICATION_JSON)
24        public Inventory getAvailability(@PathParam("itemId") String itemId) {
25 =>         Inventory inventory = em.find(Inventory.class, itemId);
26            return inventory;
27        }
28    }
~~~

Execute one line of code using `next` command so the the inventory object is 
retrieved from the database.

~~~shell
> next
~~~

Use `locals` command to see the local variables and verify the retrieved inventory 
object from the database.

~~~shell
> locals
~~~

You'll see an output similar to this.

~~~shell
default task-2[1] locals
Method arguments:
itemId = "444436"
Local variables:
inventory = null
~~~

Oh! Did you notice the problem? 

The `inventory` object which is the object retrieved from the database 
for the provided product id is `null` and is returned as the REST response! The non-existing 
product id is not a problem on its own because it simply could mean this product is discontinued 
and removed from the Inventory database but it's not removed from the product catalog database 
yet. The bug is however caused because the code returns this `null` value instead of a sensible 
REST response. If the product id does not exist, a proper JSON response stating a zero inventory 
should be returned instead of `null`.

Exit the debugger and move on to the **Fix the Inventory Bug** section to fix the bug.

~~~shell
> quit
~~~

{% endif %}

{% if REMOTE_DEBUGGER_JBDS == true %}

#### Debug Remotely Using JBoss Developer Studio (Eclipse)

JBoss Developer Studio(JBDS) is an Eclipse-bases IDE which provides a convenient way 
to debug Java applications using the Java remote debugging architecture and allows 
execute code line-by-line on a remote machine (pod in this case) while seeing 
the code within the IDE.

Start JBDS. 

If the `inventory-wildfly-swarm` project is not already imported into your 
workspace, click on **File >> Import... >> Existing Maven Projects** and then **Next**.

![Import Maven Project]({% image_path debug-jbds-import-maven.png %}){:width="600px"}

Click on **Browse**, select `inventory-wildfly-swarm` folder and click on 
**Finish**.

Open  `com.redhat.cloudnative.inventory.InventoryResource` in the code editor. Double-click 
on the editor sidebar near the first line of the `getAvailability()` 
method to add a breakpoint to that line. A circle appears near the line to show a breakpoint 
is set.

![Add Breakpoint]({% image_path debug-jbds-add-breakpoint.png %}){:width="500px"}

Now you are ready to connect to the Inventory pod. 

From the menu, click on **Run >> Debug Configurations**. The debug configurations window 
opens. From the left sidebar, double-click on **Remote Java Application** to create a new debug 
configuration for Java remote debugging. 

Set the port field to `5005` as it was forwarded to your local machine on and 
leave the rest of the fields with default values. Click on **Debug** button.

![Add Breakpoint]({% image_path debug-jbds-debug-config.png %}){:width="800px"}

JBDS connects to the Inventory pod and it's ready for debugging. Use `curl` to invoke the 
Inventory API with the suspect product id in order to pause the 
code execution at the defined breakpoint.

>  You can find out the Inventory route url using `oc get routes`. Replace 
> `{{INVENTORY_ROUTE_HOST}}` with the Inventory route url from your project.

~~~
$ curl -v http://{{INVENTORY_ROUTE_HOST}}/api/inventory/444436
~~~

JBDS switches to the *Debug Perspective* and pauses on the breakpoint.

![JBDS Debug]({% image_path debug-jbds-debug-view.png %}){:width="900px"}

Click on the step over icon to execute one line and retrieve the inventory object for the 
given product id from the database.

Can you spot the bug now? 

Look at the **Variables** window. The retrieved inventory object is `null`. 

![Debug Variables]({% image_path debug-jbds-debug-vars.png %}){:width="600px"}

You can also verify that by hovering your mouse over the `inventory` variable in the code 
editor.

![Debug Variables]({% image_path debug-jbds-debug-hover.png %}){:width="600px"}

The non-existing product id is not a problem on its own because it simply could mean 
this product is discontinued and removed from the Inventory database but it's not 
removed from the product catalog database yet. The bug is however caused because 
the code returns this `null` value instead of a sensible REST response. If the product 
id does not exist, a proper JSON response stating a zero inventory should be 
returned instead of `null`.

Stop the debugger and move on to the **Fix the Inventory Bug** section to fix the bug.

{% endif %}

{% if REMOTE_DEBUGGER_IDEA == true %}

#### Debug Remotely Using IntelliJ IDEA

IntellJ IDEA is and IDEA that among other things provides a convenient way 
to debug Java applications using the Java remote debugging architecture and allows 
execute code line-by-line on a remote machine (pod in this case) while seeing 
the code within the IDE.

Start IntellJ. 

If the `inventory-wildfly-swarm` project is not already imported into your 
workspace, click on **Import Project** and then select `inventory-wildfly-swarm` 
folder. Click on **Next** a few times and then click on **Finish**.

Open `com.redhat.cloudnative.inventory.InventoryResource` in the editor. Click on the editor 
sidebar near the first line of the `getAvailability()` method to add a breakpoint to that line. 
A circle appears near the line to show a breakpoint is set.

![Add Breakpoint]({% image_path debug-idea-add-breakpoint.png %}){:width="650px"}

From the menu, click on **Run >> Edit Configurations...** to create a new Java remote debug 
configuration. Click on the plus icon and then from the drop down list click on **Remote**

![Add Debug Configuration]({% image_path debug-idea-edit-config.png %}){:width="840px"}

In the debug configuration, specify `inventory` as name, make sure the port is `5005` and click 
on **OK**.

![Add Debug Configuration]({% image_path debug-idea-debug-config.png %}){:width="840px"}

Now you are ready to connect to the Inventory pod. From the menu, click on 
**Run >> Debug 'inventory'** to connect to the Inventory pod.

Use `curl` to invoke the Inventory API with the suspect product id in order to pause the 
code execution at the defined breakpoint.

> You can find out the Inventory route url using `oc get routes`. Replace 
> `{{INVENTORY_ROUTE_HOST}}` with the Inventory route url from your project.

~~~
$ curl -v http://{{INVENTORY_ROUTE_HOST}}/api/inventory/444436
~~~

IDEA pauses on the breakpoint.

![IntelliJS IDEA Debug]({% image_path debug-idea-debug-view.png %}){:width="900px"}

Click on the step over icon to execute one line and retrieve the inventory object for the 
given product id from the database.

Could you spot the bug now? The retrieved inventory object is `null`. 

![Variables]({% image_path debug-idea-debug-vars.png %}){:width="700px"}

The non-existing product id is not a problem on its own because it simply could mean 
this product is discontinued and removed from the Inventory database but it's not 
removed from the product catalog database yet. The bug is however caused because 
the code returns this `null` value instead of a sensible REST response. If the product 
id does not exist, a proper JSON response stating a zero inventory should be 
returned instead of `null`.

Stop the debugger and move on to the **Fix the Inventory Bug** section to fix the bug.

{% endif %}

#### Fix the Inventory Bug

Edit the `InventoryResource.java` add update the `getAvailability()` to make it look like the following 
code in order to return a zero inventory for products that don't exist in the inventory 
database:

~~~java
@GET
@Path("/api/inventory/{itemId}")
@Produces(MediaType.APPLICATION_JSON)
public Inventory getAvailability(@PathParam("itemId") String itemId) {
    Inventory inventory = em.find(Inventory.class, itemId);

    if (inventory == null) {
        inventory = new Inventory();
        inventory.setItemId(itemId);
        inventory.setQuantity(0);
    }

    return inventory;
}
~~~

Commit the changes to the Git repository.

~~~shell
$ git add src/main/java/com/redhat/cloudnative/inventory/InventoryResource.java
$ git commit -m "inventory returns zero for non-existing product id" 
$ git push origin master
~~~

As soon as you commit the changes to the Git repository, the `inventory-pipeline` gets 
triggered to build and deploy a new Inventory container with the fix. Go to the 
OpenShift Web Console and inside the **{{COOLSTORE_PROJECT}}** project. On the sidebar 
menu, Click on **Builds >> Pipelines** to see its progress.

When the pipeline completes successfully, point your browser at the Web route and verify 
that the inventory status is visible for all products. The suspect product should show 
the inventory status as _Not in Stock_.

![Inventory Status Bug Fixed]({% image_path debug-coolstore-bug-fixed.png %}){:width="800px"}

Well done and congratulations for completing all the labs.
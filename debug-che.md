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

Enable remote debugging on Inventory by running the following inside the `labs/inventory-wildfly-swarm` 
directory in the Eclipse Che **Terminal** window:

~~~shell
$ mvn fabric8:debug
~~~~

> The default port for remoting debugging is `5005` but you can change the default port 
> via environment variables. Read more in the [Java S2I Image docs](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/reference#configuration_environment_variables).

You are all set now to start debugging using the tools of you choice. 

Do not wait for the command to return! The fabric8 maven plugin keeps the forwarded 
port open so that you can start debugging remotely.

![Fabric8 Debug]({% image_path debug-che-fabric8.png %}){:width="900px"}

#### Remote Debug with Eclipse Che

Eclipse Che provides a convenience way to remotely connect to Java applications running 
inside containers and debug while following the code execution in the IDE.

From the **Run** menu, click on **Edit Debug Configurations...**.

![Remote Debug]({% image_path debug-che-debug-config-1.png %}){:width="600px"}

The window shows the debuggers available in Eclipse Che. Click on the plus sign near the 
Java debugger.

![Remote Debug]({% image_path debug-che-debug-config-2.png %}){:width="700px"}

Configure the remote debugger and click on the **Save** button:

* Check **Connect to process on workspace machine**
* Port: `5005`

![Remote Debug]({% image_path debug-che-debug-config-3.png %}){:width="700px"}

You can now click on the **Debug** button to make Eclipse Che connect to the 
Inventory service running on OpenShift.

You should see a confirmation that the remote debugger is successfully connected.

![Remote Debug]({% image_path debug-che-debug-config-4.png %}){:width="360px"}

Open `com.redhat.cloudnative.inventory.InventoryResource` and double-click 
on the editor sidebar on the line number of the first line of the `getAvailability()` 
method to add a breakpoint to that line. A start appears near the line to show a breakpoint 
is set.

![Add Breakpoint]({% image_path debug-che-breakpoint.png %}){:width="600px"}

Open a new **Terminal** window and use `curl` to invoke the Inventory API with the 
suspect product id in order to pause the code execution at the defined breakpoint.

Note that you can use the the following icons to switch between debug and terminal windows.


![Icons]({% image_path debug-che-window-guide.png %}){:width="700px"}

>  You can find out the Inventory route url using `oc get routes`. Replace 
> `{{INVENTORY_ROUTE_HOST}}` with the Inventory route url from your project.

~~~
$ curl -v http://{{INVENTORY_ROUTE_HOST}}/api/inventory/444436
~~~

Switch back to the debug panel and notice that the code execution is paused at the 
breakpoint on `InventoryResource` class.

![Icons]({% image_path debug-che-breakpoint-stop.png %}){:width="900px"}

Click on the _Step Over_ icon to execute one line and retrieve the inventory object for the 
given product id from the database.

![Step Over]({% image_path debug-che-step-over.png %}){:width="340px"}

Click on the the plus icon in the **Variables** panel to add the `inventory` variable 
to the list of watch variables. This would allow you to see the value of `inventory` variable 
during execution.

![Watch Variables]({% image_path debug-che-variables.png %}){:width="500px"}

![Debug]({% image_path debug-che-breakpoint-values.png %}){:width="900px"}

Can you spot the bug now? 

Look at the **Variables** window. The retrieved inventory object is `null`!

The non-existing product id is not a problem on its own because it simply could mean 
this product is discontinued and removed from the Inventory database but it's not 
removed from the product catalog database yet. The bug is however caused because 
the code returns this `null` value instead of a sensible REST response. If the product 
id does not exist, a proper JSON response stating a zero inventory should be 
returned instead of `null`.

Click on the _Resume_ icon to continue the code execution and then on the stop icon to 
end the debug session.

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

Go back to the **Terminal** window where `fabric8:debug` was running. Press 
`Ctrl+C` to stop the debug and port-forward and then run the following commands 
to commit the changes to the Git repository.

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
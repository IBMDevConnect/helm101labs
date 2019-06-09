# Why Helm?

[Helm](https://docs.helm.sh/) is often described as the Kubernetes application package manager. So, what does Helm give you over using kubectl directly? 

# Objectives

This lab provide an insight on the advantages of using Helm over using Kubernetes directly through `kubectl`. When you complete all the labs, you'll:
* Understand the core concepts of Helm
* Understand the advantages of deployment using Helm over Kubernetes directly, looking at:
  * Application management
  * Updates
  * Configuration
  * Revision management

# Prerequisites

* Have a running Kubernetes cluster. See the [IBM Cloud Kubernetes Service](https://cloud.ibm.com/docs/containers/cs_tutorials.html#cs_cluster_tutorial) or [Kubernetes Getting Started Guide](https://kubernetes.io/docs/setup/) for details about creating a cluster.
* Have Helm installed and initialized with the Kubernetes cluster. See [Installing Helm on IBM Cloud Kubernetes Service](Lab0/README.md) or the [Helm Quickstart Guide](https://docs.helm.sh/using_helm/#quickstart) for getting started with Helm.

# Lab 1. I just want to deploy!

Let's investigate how Helm can help us focus on other things by letting a chart do the work for us. The application is the [Guestbook App](https://github.com/IBM/guestbook), which is a sample multi-tier web application.

1. Clone the repo

```$ git clone https://github.com/IBM/helm101```

Let's go ahead and install the chart now.

2. Install the app as a Helm chart:

    ```
    $ cd helm101/charts
    $ helm install ./guestbook/ --name guestbook-demo --namespace helm-demo
    ```
    
    Note: `$ helm install` command will create the `helm-demo` namespace if it does not exist.
    
    You should see output similar to the following:
    
    ```console
    NAME:   guestbook-demo
    LAST DEPLOYED: Fri Sep 21 14:26:01 2018
    NAMESPACE: helm-demo
    STATUS: DEPLOYED
    
    RESOURCES:
    ==> v1/Service
    NAME            AGE
    guestbook-demo  0s
    redis-master    0s
    redis-slave     0s
    
    ==> v1/Deployment
    guestbook-demo  0s
    redis-master    0s
    redis-slave     0s

    ==> v1/Pod(related)
    NAME                             READY  STATUS             RESTARTS  AGE
    guestbook-demo-5dccd68c88-hqlws  0/1    ContainerCreating  0         0s
    guestbook-demo-5dccd68c88-sdhcv  0/1    ContainerCreating  0         0s
    redis-master-5d8b66464f-g9q7m    0/1    ContainerCreating  0         0s
    redis-slave-586b4c847c-ct77m     0/1    ContainerCreating  0         0s
    redis-slave-586b4c847c-nrzwj     0/1    ContainerCreating  0         0s

    NOTES:
    1. Get the application URL by running these commands:
      NOTE: It may take a few minutes for the LoadBalancer IP to be available.
            You can watch the status of by running 'kubectl get svc -w guestbook-demo --namespace helm-demo'
      export SERVICE_IP=$(kubectl get svc --namespace helm-demo guestbook-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
      echo http://$SERVICE_IP:3000
    ```
    
    The chart install performs the Kubernetes deployments and service creations of the redis master and slaves, and the guestbook app, as one. This is because the chart is a collection of files that describe a related set of Kubernetes resources and Helm manages the creation of these resources via the Kubernetes API.    
    
    To check the deployment, you can use `$ kubectl get deployment guestbook-demo --namespace helm-demo`.
    
    You should see output similar to the following:
    
    ```console
    NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    guestbook-demo   2         2         2            2           51m
    ```
    
    To check the status of the running application, you can use `$ kubectl get pods --namespace helm-demo`.
    
    ```console
    NAME                            READY     STATUS    RESTARTS   AGE
    guestbook-demo-6c9cf8b9-jwbs9   1/1       Running   0          52m
    guestbook-demo-6c9cf8b9-qk4fb   1/1       Running   0          52m
    redis-master-5d8b66464f-j72jf   1/1       Running   0          52m
    redis-slave-586b4c847c-2xt99    1/1       Running   0          52m
    redis-slave-586b4c847c-q7rq5    1/1       Running   0          52m
    ```
   
    To check the services, you can run `$ kubectl get services --namespace helm-demo`.
    
    ```console
    NAME             TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
    guestbook-demo   LoadBalancer   172.21.43.244    <pending>     3000:31367/TCP   50m
    redis-master     ClusterIP      172.21.12.43     <none>        6379/TCP         50m
    redis-slave      ClusterIP      172.21.176.148   <none>        6379/TCP         50m
    ```
    
3. View the guestbook:

   You can now play with the guestbook that you just created by opening it in a browser (it might take a few moments for the guestbook to come up).
    1. To view the guestbook on a remote host, locate the external IP and the port of the load balancer by following the "NOTES" section in the install output. The commands will be similar to the following:
    
       ```console
       $ export SERVICE_IP=$(kubectl get svc --namespace helm-demo guestbook-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
       $ echo http://$SERVICE_IP:331367
       http://50.23.5.136:31367
       ```

       In this scenario the URL is `http://50.23.5.136:31367`.
       
       Note: If no external IP is assigned, then you can get the external IP with the following command:

       ```console
       $ kubectl get nodes -o wide
       NAME           STATUS    ROLES     AGE       VERSION        EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME  
       10.47.122.98   Ready     <none>    1h        v1.10.11+IKS   173.193.92.112   Ubuntu 16.04.5 LTS   4.4.0-141-generic   docker://18.6.1
       ```

       In this scenario the URL is `http://173.193.92.112:31838`.
 
    2. Navigate to the output given (for example `http://50.23.5.136:31367`) in your browser. You should see the guestbook now displaying in your browser.

From this lab, you can see that using Helm required less commands and less to think about (by giving it the chart path and not the individual files) versus using `kubectl`. Helm's application management provides the user with this simplicity.

Move on to the next lab below to learn how to update our running app when the chart has been changed.

# Lab 2. I need to change but want none of the hassle!

In Lab 1, we installed the guestbook sample app by using Helm and saw the benefits over using `kubectl`. You probably think that you're done and know enough to use Helm. But what about updates or improvements to the chart? How do you update your running app to pick up these changes? 

In this lab, we're going to look at how to update our running app when the chart has been changed. To demonstrate this, we're going to make changes to the original `guestbook` chart by:
* Removing the Redis slaves and using just the in-membory DB
* Changing the type from `LoadBalancer` to `NodePort`.

It seems contrived but the goal of this lab is to show you how to update your apps with Helm. So, how easy is it to do this? Let's take a look below.

We'll update the previously deployed `guestbook-demo` application by using Helm.

1. Update the application:

    ```$ helm upgrade guestbook-demo ./guestbook --set redis.slaveEnabled=false,service.type=NodePort --namespace helm-demo```
    
    A Helm upgrade takes an existing release and upgrades it according to the information you provide. You should see output similar to the following:
    
    ```console
    Release "guestbook-demo" has been upgraded. Happy Helming!
    LAST DEPLOYED: Mon Sep 24 10:36:18 2018
    NAMESPACE: helm-demo
    STATUS: DEPLOYED
    
    RESOURCES:
    ==> v1/Service
    NAME            AGE
    guestbook-demo  1h
    redis-master    1h
    
    ==> v1/Deployment
    guestbook-demo  1h
    redis-master    1h
    
    ==> v1/Pod(related)
    
    NAME                           READY  STATUS   RESTARTS  AGE
    guestbook-demo-6c9cf8b9-dhqk9  1/1    Running  0         1h
    guestbook-demo-6c9cf8b9-zddn2  1/1    Running  0         1h
    redis-master-5d8b66464f-g7sh6  1/1    Running  0         1h
    
    
    NOTES:
    1. Get the application URL by running these commands:
      export NODE_PORT=$(kubectl get --namespace helm-demo -o jsonpath="{.spec.ports[0].nodePort}" services guestbook-demo)
      export NODE_IP=$(kubectl get nodes --namespace helm-demo -o jsonpath="{.items[0].status.addresses[0].address}")
      echo http://$NODE_IP:$NODE_PORT
    ```
    
    The `upgrade` command upgrades the app to a specified version of a chart, removes the `redis-slave` resources, and updates the app `service.type` to `NodePort`.
        
    To check the updates, you can run ```$ kubectl get all --namespace helm-demo```
    
    ```console
    NAME                                READY     STATUS    RESTARTS   AGE
    pod/guestbook-demo-6c9cf8b9-dhqk9   1/1       Running   0          1h
    pod/guestbook-demo-6c9cf8b9-zddn2   1/1       Running   0          1h
    pod/redis-master-5d8b66464f-g7sh6   1/1       Running   0          1h
    
    NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
    service/guestbook-demo   NodePort    172.21.43.244   <none>        3000:31202/TCP   1h
    service/redis-master     ClusterIP   172.21.12.43    <none>        6379/TCP         1h
    
    NAME                             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/guestbook-demo   2         2         2            2           1h
    deployment.apps/redis-master     1         1         1            1           1h
    
    NAME                                      DESIRED   CURRENT   READY     AGE
    replicaset.apps/guestbook-demo-6c9cf8b9   2         2         2         1h
    replicaset.apps/redis-master-5d8b66464f   1         1         1         1h
    ```
    Note: The service type has changed (to `NodePort`) and a new port has been allocated (`31202` in this output case) to the guestbook service. All `redis-slave` resources have been removed.
    
2. View the guestbook using the updated port for the guestbook service.

   ```console
       $ kubectl get nodes -o wide
       NAME           STATUS    ROLES     AGE       VERSION        EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION        CONTAINER-RUNTIME  
       10.47.122.98   Ready     <none>    1h        v1.10.11+IKS   173.193.92.112   Ubuntu 16.04.5 LTS   4.4.0-141-generic   docker://18.6.1
       ```

       In this scenario the URL is `http://173.193.92.112:31202`.

Congratulations, you have now updated the applications! Helm does not require any manual changing of resources and is therefore so much easier to upgrade! All configurations can be set on the fly on the command line or by using override files. This is made possible from when the logic was added to the template files, which enables or disables the capability, depending on the flag set.

# Lab 3. Keeping track of the deployed application

Let's say you deployed different release versions of your application (i.e., you upgraded the running application). How do you keep track of the versions and how can you do a rollback?

In this part of the lab, we illustrate revision management on the deployed application `guestbook-demo` by using Helm.

With Helm, every time an install, upgrade, or rollback happens, the revision number is incremented by 1. The first revision number is always 1. Helm persists release metadata in configmaps, stored in the Kubernetes cluster. Every time your release changes, it appends that to the existing data. This provides Helm with the capability to rollback to a previous release.

Let's see how this works in practice.

1. Check the number of deployments:

    ```console
    $ helm history guestbook-demo
    ```
    
    You should see output similar to the following because we did an upgrade in [Lab 2](../Lab2/README.md) after the initial install in [Lab 1](../Lab1/README.md):
    
    ```console
    REVISION	UPDATED                 	STATUS    	CHART          	DESCRIPTION
    1       	Mon Sep 24 08:54:04 2018	SUPERSEDED	guestbook-0.1.0	Install complete
    2       	Mon Sep 24 10:36:18 2018	DEPLOYED  	guestbook-0.1.0	Upgrade complete
    ```
        
2. Roll back to the previous revision:

    In this rollback, Helm checks the changes that occured when upgrading from the revision 1 to revision 2. This information enables it to makes the calls to the Kubernetes API server, to update the deployed application as per the initial deployment - in other words with Redis slaves and using a load balancer.

    Rollback with this command, ```$ helm rollback guestbook-demo 1```
    
    ```console
    Rollback was a success! Happy Helming!
    ```
    Check the history again, `$ helm history guestbook-demo`
    
    You should see output similar to the following:
    
    ```console
    REVISION	UPDATED                 	STATUS    	CHART          	DESCRIPTION     
    1       	Mon Sep 24 08:54:04 2018	SUPERSEDED	guestbook-0.1.0	Install complete
    2       	Mon Sep 24 10:36:18 2018	SUPERSEDED	guestbook-0.1.0	Upgrade complete
    3       	Mon Sep 24 11:59:18 2018	DEPLOYED  	guestbook-0.1.0	Rollback to 1
    ```
    
    To check the rollback, you can run `$ kubectl get all --namespace helm-demo`:
    
    ```console
    NAME                                READY     STATUS    RESTARTS   AGE
    pod/guestbook-demo-6c9cf8b9-dhqk9   1/1       Running   0          3h
    pod/guestbook-demo-6c9cf8b9-zddn2   1/1       Running   0          3h
    pod/redis-master-5d8b66464f-g7sh6   1/1       Running   0          3h
    pod/redis-slave-586b4c847c-tkfj5    1/1       Running   0          2m
    pod/redis-slave-586b4c847c-xxrdn    1/1       Running   0          2m
    
    NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
    service/guestbook-demo   LoadBalancer   172.21.43.244   <pending>     3000:31202/TCP   3h
    service/redis-master     ClusterIP      172.21.12.43    <none>        6379/TCP         3h
    service/redis-slave      ClusterIP      172.21.232.16   <none>        6379/TCP         2m
    
    NAME                             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/guestbook-demo   2         2         2            2           3h
    deployment.apps/redis-master     1         1         1            1           3h
    deployment.apps/redis-slave      2         2         2            2           2m
    
    NAME                                      DESIRED   CURRENT   READY     AGE
    replicaset.apps/guestbook-demo-6c9cf8b9   2         2         2         3h
    replicaset.apps/redis-master-5d8b66464f   1         1         1         3h
    replicaset.apps/redis-slave-586b4c847c    2         2         2         2m
    ```
   
    You can see from the output that the app service is the service type of `LoadBalancer` again and the Redis master/slave deployment has returned. 
    This shows a complete rollback from the upgrade in Lab 2 

From this lab, we can say that Helm does revision management well and Kubernetes does not have the capability built in! You might be wondering why we need `helm rollback` when you could just re-run the `helm upgrade` from a previous version. And that's a good question. Technically, you should end up with the same resources (with same parameters) deployed. However, the advantage of using `helm rollback` is that helm manages (ie. remembers) all of the variations/parameters of the previous `helm install\upgrade` for you. Doing the rollback via a `helm upgrade` requires you (and your entire team) to manually track how the command was previously executed. That's not only tedious but very error prone. It is much easier, safer and reliable to let Helm manage all of that for you and all you need to do it tell it which previous version to go back to, and it does the rest.

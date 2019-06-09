# Why Helm?

[Helm](https://docs.helm.sh/) is often described as the Kubernetes application package manager. So, what does Helm give you over using kubectl directly? 

# Objectives

These labs provide an insight on the advantages of using Helm over using Kubernetes directly through `kubectl`. When you complete all the labs, you'll:
* Understand the core concepts of Helm
* Understand the advantages of deployment using Helm over Kubernetes directly, looking at:
  * Application management
  * Updates
  * Configuration
  * Revision management
  * Repositories and chart sharing

# Prerequisites

* Have a running Kubernetes cluster. See the [IBM Cloud Kubernetes Service](https://cloud.ibm.com/docs/containers/cs_tutorials.html#cs_cluster_tutorial) or [Kubernetes Getting Started Guide](https://kubernetes.io/docs/setup/) for details about creating a cluster.
* Have Helm installed and initialized with the Kubernetes cluster. See [Installing Helm on IBM Cloud Kubernetes Service](Lab0/README.md) or the [Helm Quickstart Guide](https://docs.helm.sh/using_helm/#quickstart) for getting started with Helm.

# Lab 1. I just want to deploy!

Let's investigate how Helm can help us focus on other things by letting a chart do the work for us. The application is the [Guestbook App](https://github.com/IBM/guestbook), which is a sample multi-tier web application.





From this lab, you can see that using Helm required less commands and less to think about (by giving it the chart path and not the individual files) versus using `kubectl`. Helm's application management provides the user with this simplicity.

Move on to the next lab below to learn how to update our running app when the chart has been changed.

# Lab 2. I need to change but want none of the hassle

In [Lab 1], we installed the guestbook sample app by using Helm and saw the benefits over using `kubectl`. You probably think that you're done and know enough to use Helm. But what about updates or improvements to the chart? How do you update your running app to pick up these changes? 

In this lab, we're going to look at how to update our running app when the chart has been changed. To demonstrate this, we're going to make changes to the original `guestbook` chart by:
* Removing the Redis slaves and using just the in-membory DB
* Changing the type from `LoadBalancer` to `NodePort`.

It seems contrived but the goal of this lab is to show you how to update your apps with Kubernetes and Helm. So, how easy is it to do this? Let's take a look below.

We'll update the previously deployed `guestbook-demo` application by using Helm.

Before we start, let's take a few minutes to see how Helm simplifies the process compared to using Kubernetes directly. Helm's use of a [template language](https://docs.helm.sh/chart_template_guide/) provides great flexibility and power to chart authors, which removes the complexity to the chart user. In the guestbook example, we'll use the following capabilities of templating:
* Values: An object that provides access to the values passed into the chart. An example of this is in `guestbook-service`, which contains the line `type: {{ .Values.service.type }}`. This line provides the capability to set the service type during an upgrade or install.
* Control structures: Also called “actions” in template parlance, control structures provide the template author with the ability to control the flow of a template’s generation. An example of this is in `redis-slave-service`, which contains the line `{{- if .Values.redis.slaveEnabled -}}`. This line allows us to enable/disable the REDIS master/slave during an upgrade or install.

The complete `redis-slave-service.yaml` file shown below, demonstrates how the file becomes redundant when the `slaveEnabled` flag is disabled and also how the port value is set. There are more examples of templating functionality in the other chart files. 

```
{{- if .Values.redis.slaveEnabled -}}
apiVersion: v1
kind: Service
metadata:
  name: redis-slave
  labels:
    app: redis
    role: slave
spec:
  ports:
  - port: {{ .Values.redis.port }}
    targetPort: redis-server
  selector:
    app: redis
    role: slave
{{- end }}
```

Enough talking about the theory. Now let's give it a go!

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
    
2. View the guestbook as per [Lab1](../Lab1/README.md), using the updated port for the guestbook service.

Congratulations, you have now updated the applications! Helm does not require any manual changing of resources and is therefore so much easier to upgrade! All configurations can be set on the fly on the command line or by using override files. This is made possible from when the logic was added to the template files, which enables or disables the capability, depending on the flag set.

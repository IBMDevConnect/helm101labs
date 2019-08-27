# Why Helm?

[Helm](https://docs.helm.sh/) is often described as the Kubernetes application package manager. So, what does Helm give you over using kubectl directly? 

# Objectives

This lab provide an insight on the advantages of using Helm over using Kubernetes directly through `kubectl`. When you complete all the labs, you'll:
* Understand the core concepts of Helm
* Understand the advantages of deployment using Helm over Kubernetes directly, looking at:
  * Application management

# Prerequisites

* Have a running Kubernetes cluster. See the [IBM Cloud Kubernetes Service](https://cloud.ibm.com/docs/containers/cs_tutorials.html#cs_cluster_tutorial) or [Kubernetes Getting Started Guide](https://kubernetes.io/docs/setup/) for details about creating a cluster.

### Install Helm

```
$ wget https://get.helm.sh/helm-v2.14.1-linux-amd64.tar.gz
$ tar -zxvf helm-v2.14.1-linux-amd64.tar.gz
$ ./linux-amd64/helm

$ helm init
$ helm version
```

# Lab 1. I just want to deploy!

Let's investigate how Helm can help us focus on other things by letting a chart do the work for us. The application is the [Guestbook App](https://github.com/IBM/guestbook), which is a sample multi-tier web application.

1. Clone the repo

```$ git clone https://github.com/IBMDevConnect/helm101```

2. Install the app as a Helm chart:

    ```
    $ cd helm101/tiller/
    $ kubectl apply -f deployment.yaml
    $ kubectl apply -f service.yaml
    
    $ kubectl create serviceaccount --namespace kube-system tiller
    $ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
    $ kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
    
    $ kubectl create namespace helm-demo
    
    Go back to $ cd linux-amd64
    
    $ ./helm install ../helm101/charts/guestbook/ --generate-name --namespace helm-demo
    ```
    
    You should see output similar to the following:
    
    ```
    NAME: guestbook-1566891137
    LAST DEPLOYED: 2019-08-27 07:32:18.005602971 +0000 UTC m=+0.276023370
    NAMESPACE: helm-demo
    STATUS: deployed
    NOTES:
    1. Get the application URL by running these commands:
    NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        You can watch the status of by running 'kubectl get svc -w guestbook-1566891137 --namespace helm-demo'
   export SERVICE_IP=$(kubectl get svc --namespace helm-demo guestbook-1566891137 -o   jsonpath='{.status.loadBalancer.ingress[0].ip}')
   echo http://$SERVICE_IP:3000
    ```
    
    The chart install performs the Kubernetes deployments and service creations of the redis master and slaves, and the guestbook app, as one. This is because the chart is a collection of files that describe a related set of Kubernetes resources and Helm manages the creation of these resources via the Kubernetes API.    
    
    To check the deployment, you can use `$ kubectl get deployment REPLACE_WITH_GUESTBOOK_NAME --namespace helm-demo`.
    
    You should see output similar to the following:
    
    ```console
    NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    guestbook-1566891137   2         2         2            2           51m
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
    
    **Take a note of the guestbook-******* PORT** For example, 31367 is the port here.
    
3. View the guestbook:

   You can now play with the guestbook that you just created by opening it in a browser (it might take a few moments for the guestbook to come up).
    
       ```console
       $ kubectl get nodes -o wide
       NAME           STATUS    ROLES     AGE       VERSION        EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME  
       10.47.122.98   Ready     <none>    1h        v1.10.11+IKS   173.193.92.112   Ubuntu 16.04.5 LTS   4.4.0-141-generic   docker://18.6.1
       ```

       In this scenario the URL is `http://173.193.92.112:31367`.
 
    2. Navigate to the output given (for example `http://50.23.5.136:31367`) in your browser. You should see the guestbook now displaying in your browser.

![Guestbook](https://github.com/IBM/helm101/blob/master/tutorial/images/guestbook-page.png)

From this lab, you can see that using Helm required less commands and less to think about (by giving it the chart path and not the individual files) versus using `kubectl`. Helm's application management provides the user with this simplicity.

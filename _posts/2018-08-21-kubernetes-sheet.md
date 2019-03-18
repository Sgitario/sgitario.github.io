---
layout: post
title: Kubernetes Sheet
date: 2018-08-21
tags: [ Kubernetes ]
---

Credits to [Udemy course](https://www.udemy.com/kubernetes-from-a-devops-kubernetes-guru/).
Github sources [here](https://github.com/Sgitario/kubernetes-demo).
Cheat sheet by [Kubernetes official](https://kubernetes.io/docs/reference/kubectl/cheatsheet/).

h1. Introduction

Let's give a very quick overview of main kubernetes concepts with some useful commands that really helped me a lot.

h2. Pods

Kubernetes works on Pods basis. A pod is a set of co-related containers. The most common scenario is running a single container per Pod. We can [label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) pods or add pods in [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) to better organise them in logical units. 

```bash
kubectl create -f pod.yaml
kubectl get pods // get all the pods
kubectl get pods -l app=collector // get all pods with a label called app with value "collector"
kubectl describe <name> // get pod information like IP or volumes or **events**
kubectl exec -it <name> bash // enter to the pod system and then we can see the containers inside
kubectl logs -f <name> -c <containername> // see the logs of a container running in a pod
kubectl delete pods --all // delete all the pods
```

h2. Deployments

The pods are un-managed resources which means that if a pod is dead, Kubernete will do nothing to recover it back. In order to manage pods, we need to use **ReplicationControllers** or **ReplicaSets**. 

Don't want to add more details about these two resource types since we'll be using **Deployment** resources for managing pods and more things.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: collector
spec:
  replicas: 3
  selector:
    matchLabels:
      app: collector
  template:
    metadata:
      labels:
        app: collector
    spec:
      containers:
      - name: collector
        image: dockercontainer:tag
        ports:
        - containerPort: 8080
```

When we apply this deployment, Kubernetes will create one or more **ReplicaSets** resources to manage the pods by us. We can manage the number of pods we want to create using the **replicas** field. Easy peasy!

Moreover, the **Deployment** resource can offer much more! It works like if we tag our branch in git, but with a whole infrastructure. We can tag the state of a whole infrastructure and rollback whenever we want. 

And eventually, we'll want to upgrade our applications. This resource manages the upgrade of existing containers using two strategies: 
- StrategyType: RollingUpdate. This is the default strategy. We'll update all the pods gradually. To be used if our stack supports multiple different versions running at the same time. 
- StrategyType: Recreate. It will delete all the existing pods and recreate them with the new version. 

This is done transparently by Kubernetes when we update a Deployment template.

h3. Using Private Docker Repository

This only works for docker images that are hosted in [dockerhub](https://hub.docker.com/). Most of the cases, we want to use a private docker repository. Let's see how to configure this. We need a **Secret** resource (we'll see more about this resource later in this tutorial) to provide the docker credentials:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dockerconfigjson
data:
  .dockerconfigjson: content of your $HOME/.docker/config.json
type: kubernetes.io/dockerconfigjson
```

And use it in your deployment template like:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: collector
spec:
  replicas: 3
  selector:
    matchLabels:
      app: collector
  template:
    metadata:
      labels:
        app: collector
    spec:
      containers:
      - name: collector
        image: dockercontainer:tag
        ports:
        - containerPort: 8080
      imagePullSecrets:
      - name: dockerconfigjson
```

h3. Resources
Another important topic when deploying our apps is the server [resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) we're using or we'll need:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

The *resources.requests* field works like a minimum resources the application needs to work well. The *resources.limits* is the maximum resources that the application should consume.

h3. Pods Ordering

Sometimes, our apps need some services to start working, how can we deal with pods startup orderning? Let's introduce the init containers:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-poststart-hook
spec:
  initContainers:
  - name: init
    image: busybox
    command:
    - sh
    - -c
     - 'while true; do echo "Waiting for a service to come up..."; ➥ wget http://myservice -q -T 1 -O /dev/null >/dev/null 2>/dev/null ➥ && break; sleep 1; done; echo "Service is up! Starting main
➥ container."'
  containers:
  - image: ...
    name: ...
```

h2. Services

How do we expose our endpoints? We need to use another resource called **Services**:
- The default service type is "ClusterIP" which will expose the pods only within current node. 
- The service type "NodePort" will expose the pods within the same cluster. 
- Setting the service type to "LoadBalancer" will expose the pods outside too via public ip. 

```yaml
kind: Service
apiVersion: v1
metadata:
  name: collectorservice
spec:
  type: NodePort
  selector:
    app: myApp
  ports:
  - name: myAppRest
    port: 8180
```

This is an example of how to create a service in Kubernetes. In Minikube, we can test this service by doing:

```bash
minikube service collectorservice
```

Some useful commands about services:
```bash
kubectl get services
```

All the resources we have introduced so far: **ReplicationControllers**, **ReplicaSets** and **Services**, select the pods via label selection. This is if our pod is labelled with "app=myApp", we can configure our **Service** to scope the pods with label "app=myApp" only. 

h2. Volumes

Usually, our applications need paths to work. Either a folder where to put the logging files that eventually will be read by a third system like logstash or splunk. Or a folder where to locate temporal files we need to watch or import on demand. Or a folder where to read the configuration files from. 

Kubernetes provides the **Volume** resources for this purpose. And there are MANY type of volumens you can mount. We can find the complete list of volume types [here](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes).

Let's give a very easy example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: collector
spec:
  replicas: 3
  selector:
    matchLabels:
      app: collector
  template:
    metadata:
      labels:
        app: collector
    spec:
      containers:
      - name: collector
        image: dockercontainer:tag
        ports:
        - containerPort: 8080
        volumeMounts:
        - mountPath: /tmp/logs //container path
          name: logdata //volume name ref
      volumes:
      - name: logdata //volume name
        emptyDir: {} //this is the volume type
```

The volume type *emptyDir* will create an empty folder. Then we'll mount this empty folder in our containers. We could map the same volume to different containers and these containers could send files each folder through this volume. 

Bear in mind, our pods should be node agnostic, so we should **NEVER* mount local folders in. This is a very bad practice. This is because the pod could work differently if it's deployed in a node where the data in its local folder is different from another node. 

h2. Configuration

What about the configuration files? Do we need to upload these files into a volume in the cloud like S3 always for kubernetes? **NO**. Kubernetes provides a couple of special volumes for you.

h3. ConfigMap

Let's see how looks like:
```yaml
apiVersion: v1
data:
  config.properties: |
    environment=app
  settings.xml: |
    <xml>
    <value>something</value>
    </xml>
kind: ConfigMap
metadata:
  name: my-app-config
```
We added a **ConfigMap** resource with a couple of files. Let's use it:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: collector
spec:
  replicas: 3
  selector:
    matchLabels:
      app: collector
  template:
    metadata:
      labels:
        app: collector
    spec:
      containers:
      - name: collector
        image: dockercontainer:tag
        ports:
        - containerPort: 8080
        volumeMounts:
        - mountPath: /opt/config //container path
          name: config //volume name ref
      volumes:
      - name: config //volume name
        configMap:
          name: my-app-config
```

h3. Secrets

For the secrets, it's almost the same as in ConfigMaps but it works differently internally. We edit the template using Base64 and this configuration is not stored in each node, but it keeps in memory. More about this in [here](https://kubernetes.io/docs/concepts/storage/volumes/#secret).

h2. Dashboard Web UI

```bash
kubectl proxy
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

Or:
```bash
minikube dashboard
```

h2. Appedix: More Commands
h3. Scaling

```bash
kubectl scale —replicas=4 deployment/tomcat-deployment 
kubectl expose deployment tomcat-deployment --type=NodePort
kubectl expose deployment tomcat-deployment —type=LoadBalancer —port=8080 —target-port=8080 —name tomcat-load-balancer
kubectl describe services tomcat-load-balancer
kubectl describe services tomcat-load-balancer
```

h3. Upgrading

```bash
kubectl get deployments
kubectl rollout status
kubectl set image
kubectl rollout history
```

h3. Secrets Commands

```bash
kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
kubectl create secret generic mysql-pass --from-literal=password=PASSWORDS_IN_PLAIN_TEXT_ARE_BAD_WE_WILL_SHOW_SOMETHING_MORE_SECURE_LATER
kubectl get secret
 
kubectl apply -f mysql-deployment.yaml
kubectl apply -f wordpress-deployment.yaml
minikube service wordpress --url
```

h3. Usage & Resource Monitoring

```bash
minikube addons enable heapster
kubectl get pods --namespace=kube-system
minikube addons open heapster
```

h3. Namespaces & Resource Quotas 

```bash
kubectl create namespace <namespace name>
kubectl get namespace
kubectl create namespace cpu-limited-tomcat
kubectl create -f ./cpu-limits.yaml —namespace=cpu-limited-tomcat (from the GitHub repo directory for this lecture)
kubectl apply -f ./tomcat-deployment.yaml —namespace=cpu-limited-tomcat (from the GitHub repo directory for this lecture)
kubectl describe deployment tomcat-deployment —namespace=cpu-limited-tomcat
```

h3. Auto-Scaling

```bash
-- Terminal 1
minikube addons enable metrics-server  
kubectl apply -f ./wordpress-deployment.yaml
kubectl autoscale deployment wordpress --cpu-percent=50 --min=1 --max=5

-- Terminal 2
kubectl run -i --tty load-generator --image=busybox /bin/sh
while true; do wget -q -O- http://wordpress.default.svc.cluster.local; done

-- Terminal 1
kubectl get hpa
```

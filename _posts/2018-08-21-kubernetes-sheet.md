---
layout: post
title: Kubernetes Sheet
date: 2018-08-21
tags: [ Kubernetes ]
---

Sheet from [Udemy course](https://www.udemy.com/kubernetes-from-a-devops-kubernetes-guru/).
Github sources [here](https://github.com/Sgitario/kubernetes-demo).

h1. Basic

```bash
kubectl get pods
kubectl get pods [pod name]
kubectl expose <type name> <identifier/name> [—port=external port] [—target-port=container-port [—type=service-type]
kubectl port-forward <pod name> [LOCAL_PORT:]REMOTE_PORT]
kubectl attach <pod name> -c <container>
kubectl exec [-it] <pod name> [-c CONTAINER] — COMMAND [args…]
kubectl label [—overwrite] <type> KEY_1=VAL_1 ….
kubectl run <name> —image=image
```

h1. Scaling

```bash
kubectl scale —replicas=4 deployment/tomcat-deployment 
kubectl expose deployment tomcat-deployment --type=NodePort
kubectl expose deployment tomcat-deployment —type=LoadBalancer —port=8080 —target-port=8080 —name tomcat-load-balancer
kubectl describe services tomcat-load-balancer
kubectl describe services tomcat-load-balancer
```

h1. Upgrading

```bash
kubectl get deployments
kubectl rollout status
kubectl set image
kubectl rollout history
```

h1. Dashboard Web UI

```bash
kubectl proxy
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

Or:
```bash
minikube dashboard
```

h1. Secrets Commands

```bash
kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
kubectl create secret generic mysql-pass --from-literal=password=PASSWORDS_IN_PLAIN_TEXT_ARE_BAD_WE_WILL_SHOW_SOMETHING_MORE_SECURE_LATER
kubectl get secret
 
kubectl apply -f mysql-deployment.yaml
kubectl apply -f wordpress-deployment.yaml
minikube service wordpress --url
```

h1. Usage & Resource Monitoring

```bash
minikube addons enable heapster
kubectl get pods --namespace=kube-system
minikube addons open heapster
```

h1. Namespaces & Resource Quotas 

```bash
kubectl create namespace <namespace name>
kubectl get namespace
kubectl create namespace cpu-limited-tomcat
kubectl create -f ./cpu-limits.yaml —namespace=cpu-limited-tomcat (from the GitHub repo directory for this lecture)
kubectl apply -f ./tomcat-deployment.yaml —namespace=cpu-limited-tomcat (from the GitHub repo directory for this lecture)
kubectl describe deployment tomcat-deployment —namespace=cpu-limited-tomcat
```

h1. Auto-Scaling

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

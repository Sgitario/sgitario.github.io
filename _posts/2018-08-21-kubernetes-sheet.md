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
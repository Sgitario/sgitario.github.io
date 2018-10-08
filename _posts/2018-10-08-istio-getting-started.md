---
layout: post
title: Istio Getting Started
date: 2018-10-08
tags: [ Kubernetes, Istio ]
---

This is a very simple introduction for Istio Service Mesh. 

# Introduction

What is Istio? What is different from Kubernetes? Istio is the second evolution to microservices architectures. So far, all the solutions need to solve by its own the exactly same paradigms for service discovery, logging, configuration and a long etc. Istio is a service mesh that provides all these services for all your microservices. How to upgrade microservices, listen for the traces, routing, etc... All of this is now delegated to the Service Mesh. Istio uses kubernetes to deploy all its services.

## Some Technical Context

Istio has been developed by Google and other parties. For all your services, Istio adds a proxy [Envoy](https://www.envoyproxy.io/] inside the pod to sync with Istio

# Installation

We followed the instructions from [here](https://istio.io/docs/setup/kubernetes/quick-start/). This is:

1. Download and Install:

```bash
curl -L https://git.io/getLatestIstio | sh -
cd istio-1.0.2
export PATH=$PWD/bin:$PATH
````

2. Install [Istio's Custom Resource Definition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions):

```bash
kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
```

3. Install Istio Services:

```bash
kubectl apply -f install/kubernetes/istio-demo.yaml
```

This script is to enable Istio without auth. There are more options. See the getting started page for more. 

4. Verify Installation:

Go to Kubernetes dashboard (minikube dashboard) and select the Istio-System namespace. We should see all the Istio services in the Deployment section.

h1. Let's Write Our First App in Istio!

This is a hello world node.js app:

```js
var http = require('http');

var handleRequest = function(request, response) {
  console.log('Received request for URL: ' + request.url);
  response.writeHead(200);
  response.end('Hello World!');
};
var www = http.createServer(handleRequest);
www.listen(8080);
```

If you run this node app, we can try the app just typing "node server.js" and going to localhost:8080.

Then, we need to build the docker image:

```
FROM node:6.14.2
EXPOSE 8080
COPY server.js .
CMD node server.js
```

And install the image inside the kubernetes docker repository:

```bash
eval $(minikube docker-env)
docker build -t hello-node:v1 .
```

Finally, let's deploy our application in Istio via this deployments.yaml:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld-service
  labels:
    app: helloworld-service
spec:
  type: NodePort
  ports:
  - port: 8080
    name: http
  selector:
    app: helloworld-service
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: helloworld-service
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: helloworld-service
        version: v1
    spec:
      containers:
      - name: helloworld-service
        image: hello-node:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
```

And typing:

```
kubectl label namespace myappnamespace istio-injection=enabled
kubectl create -n myappnamespace -f deployments.yaml
```

The app should have started up now in localhost:8080.

# What Can We do Now?

- We can install plugins or addons to Istio like [Grafana](https://grafana.com/), [Zipkin](https://zipkin.io/), [Prometheus](https://prometheus.io/)... See more in [here](https://github.com/saturnism/istio-by-example-java/tree/master/spring-boot-example). All these components will see our apps by doing nothing. 

- We can define rules for routing or define test strategies for failures, just defining RouteRule in route-to-v2.yaml:

```yaml
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: helloworld-service
spec:
  destination:
    name: helloworld-service
  precedence: 1
  match:
    request:
      headers:
        user-agent:
          regex: ".*Chrome.*"
  route:
  - labels:
      version: "v2"
```

This script will route all the requests made from Chrome for our hello world app to a new version "v2" for the same app. 

- Or simulate HTTP failures:

```yaml
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: helloworld-service-503
spec:
  destination:
    name: helloworld-service
  precedence: 1
  route:
  - labels:
      version: "v1"
  httpFault:
    abort:
      percent: 100
      httpStatus: 503
```

This script will make our application to return a 503 HTTP error the 100% times.
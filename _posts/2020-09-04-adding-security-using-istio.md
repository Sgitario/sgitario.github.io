---
layout: post
title: Adding Security using Istio
date: 2020-09-04
tags: [ Keycloak, Kubernetes, Istio ]
---

In this post, we're going to use Istio to enable security to our applications deployed in the cloud (using K8S or Openshift).

The tools I'm using as part of this post are:
- [Istio 1.6.7](https://istio.io/) to enable service mesh capabilities. The Istio 1.7+ does not work with the OIDC filter that we install in section 5.
- [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) to have a local Kubernates environment
- [Keycloak](https://www.keycloak.org/) as authentication server
- [WebAssembly](https://istio.io/latest/blog/2020/wasm-announce/) to extend Istio capabilities

As you might already know, Istio enables some required service mesh capabilities to our architecture such as monitoring, traffic management and authorization. This is achieved thanks to the [sidecar proxy](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar) pattern. Which means that Istio will inject an istio sidecar container in the pods of our application to manage/proxy it.

## 0.- Configure Minikube

| Skip this step if you have already access to a cluster environment

As I said, I'm using minikube, so what I did was:

```sh
> minikube start --addons ingress --addons dashboard
```

## 1.- Install Istio

| Skip this step if you have already installed istio. 

Please, follow the instructions from [the official Istio documentation](https://istio.io/latest/docs/setup/install/).

What I did was to install the [istioctl](https://preliminary.istio.io/latest/docs/ops/diagnostic-tools/istioctl/) command line:

```sh
> curl -sL https://istio.io/downloadIstioctl | sh -
> export PATH=$PATH:$HOME/.istioctl/bin
```

Then, install the Istio operator:

```sh
> istioctl operator init --tag 1.6.7
```

And deploy our Istio instance:

```sh
> kubectl create ns istio-system
> kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: demo
EOF
```

## 2.- Install Keycloak

| Skip this step if you're have an up and running Keycloak instance or you prefer to install keycloak via the operators hub.

Please, follow the instructions from [the official Keycloak documentation](https://www.keycloak.org/getting-started/getting-started-kube) to install keycloak.

What I did was to install the keycloak deployment and service doing:

```sh
> kubectl create -f https://raw.githubusercontent.com/keycloak/keycloak-quickstarts/latest/kubernetes-examples/keycloak.yaml
```

And exposing the keycloak service via ingress:

```sh
> wget -q -O - https://raw.githubusercontent.com/keycloak/keycloak-quickstarts/latest/kubernetes-examples/keycloak-ingress.yaml | \
sed "s/KEYCLOAK_HOST/keycloak.$(minikube ip).nip.io/" | \
kubectl create -f -
```

And check the keycloak admin console in "https://keycloak.$(minikube ip).nip.io/auth".

Let's now configure the client in Keycloak:

1. Get the Keycloak token for admin rights:

```sh
> export TKN=$(curl -X POST --insecure https://keycloak.$(minikube ip).nip.io/auth/realms/master/protocol/openid-connect/token \
 -H "Content-Type: application/x-www-form-urlencoded" \
 -d "username=admin" \
 -d 'password=admin' \
 -d 'grant_type=password' \
 -d 'client_id=admin-cli' | jq -r '.access_token')
```

2. Create the client called "test"

```sh
> curl -X POST --insecure https://keycloak.$(minikube ip).nip.io/auth/admin/realms/master/clients \
 -H "authorization: Bearer $TKN" \
 -H "Content-Type: application/json" \
 --data \
 '{
    "id": "test",
    "name": "test",
    "redirectUris": ["*"]
 }' 
```

3. Get the client secret:

```sh
export CLIENT_SECRET=$(curl --insecure https://keycloak.$(minikube ip).nip.io/auth/admin/realms/master/clients/test/client-secret \
 -H "authorization: Bearer ${TKN}" \
 -H "Content-Type: application/json" | jq -r '.value')
```

**Copy the client secret as we'll need it in section 5.**

## 3.- Deploy my application without security

I'm going to use one simple "Hello World" node.js application that I've published in my quay.io account: [quay.io/jcarvaja/node-hello-world](quay.io/jcarvaja/node-hello-world).

```sh
> kubectl create ns my-app
> kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: hello-node
  namespace: my-app
  labels:
    app: hello-node
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  selector:
    app: hello-node
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-node
  namespace: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-node
  template:
    metadata:
      labels:
        app: hello-node
    spec:
      containers:
        - name: node-hello-world
          image: quay.io/jcarvaja/node-hello-world
          imagePullPolicy: Always
      restartPolicy: Always
EOF
```

At the moment, our service is not exposed externally (via routes in Openshift or ingress in K8S). Let's see now how to access our service via the Istio gateway.

## 4.- Enabling Security in our application

As you can see, our application returns the expected "Hello World!". Let's enable the security, so only authenticated users can see this message.

First, we need to instruct Istio to inspect my environment. See [the official Istio documentation](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection) for futher details.

```sh
> kubectl label namespace my-app istio-injection=enabled
```

Now, we need to delete the pods of our app, so Istio can inject the new changes:

```sh
> kubectl delete pod -l app=hello-node -n my-app
```

After some seconds, Kubernetes will recreate this pod and now we should see that there are now two containers inside it:

```sh
> kubectl describe pod -l app=hello-node -n my-app
// we should see the "node-hello-world" and the "istio-proxy" containers now! :)
``` 

| If you are interested in what happened, go to [the documentation](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#understanding-what-happened) for further details... It's really interesting!

Next, we need to configure the Istio gateway accordingly:

```sh
> kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-app-gateway
  namespace: my-app
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app-virtual-service
  namespace: my-app
spec:
  hosts:
  - "*"
  gateways:
  - my-app-gateway
  http:
  - match:
    - uri:
        prefix: /auth
    route:
    - destination:
        port:
          number: 8080
        host: keycloak.default.svc.cluster.local
  - route:
    - destination:
        port:
          number: 8080
        host: hello-node.my-app.svc.cluster.local
EOF
```

Then, if we browse the istio gateway to connect to our app using port forwarding:

```sh
> kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80
```

We should now see the "Hello World!" message in [http://localhost:8080](http://localhost:8080).

What now? We have not enabled security in our application so far, we just installed istio in our application, so let's enable the security to our app next:

```sh
> kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata: 
  name: jwt-my-app-auth
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: hello-node
  jwtRules:
  - issuer: http://localhost:8080/auth/realms/master
    jwksUri: http://keycloak.default.svc.cluster.local:8080/auth/realms/master/protocol/openid-connect/certs
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata: 
  name: frontend-my-app-policy
  namespace: my-app
spec:
  selector: 
    matchLabels: 
      app: hello-node
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
EOF
```

| More information about the RequestAuthentication in [Istio documentation](https://istio.io/latest/docs/reference/config/security/request_authentication/).

As you can see, all what we have done is to configure the Istio gateway to secure all the endpoints (including the "my-app" path). Therefore, when we browse again our app endpoint in [http://localhost:8080/my-app](http://localhost:8080/my-app), we should see now "RBAC: access denied".

## 5.- Extend Istio to handle authorization

In order to make the endpoint work, we have to send a token issued by our keycloak instance. But what about if we could make Istio to redirect the URLs to keycloak directly? There are a few ways to perform this such as [App Identity and Access Adapter from IBM](https://istio.io/latest/blog/2019/app-identity-and-access-adapter/), [the AuthService custom external implementation](https://github.com/istio-ecosystem/authservice) and [Envoy Ext_Authz plugin](https://medium.com/google-cloud/envoy-external-authorization-server-envoy-ext-authz-helloworld-82eedc7f8122).

In this port, we need to extend Istio sidebar with a custom Envoy filter using [WebAssembly](https://istio.io/latest/blog/2020/wasm-announce/) framework. At the moment, we can implement this custom filter using C++, RUST or Golang. The good news is that [Daniel Grimm](https://github.com/dgn) already implemented this custom filter called [OIDC filter](https://github.com/dgn/oidc-filter).

| Note that this OIDC filter is for development purposes only and requires some effort to make it production ready. Feel free to contribute here.

- Requirements

**Rustup:**

```sh
> curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

**Wasm Pack Cargo:**

```sh
> cargo install wasm-pack
```

**WebAssembly:**

```sh
> curl -sL https://run.solo.io/wasme/install | sh
```

- Build

```sh
> git clone https://github.com/dgn/oidc-filter
> cd oidc-filter
> rustup target add wasm32-unknown-unknown
> make build
```

- Installation

The goal is to copy the built oidc filter to be accessed by the istio-proxy container of all our pods. There are multiple approaches to follow here, we'll make use of the istio annotations "sidecar.istio.io/userMount" and "sidecar.istio.io/userVolume", and we'll create an empty folder and then copy the oidc filter to the container directly:

```sh
> kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-node
  namespace: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-node
  template:
    metadata:
      annotations:
        sidecar.istio.io/componentLogLevel: "wasm:info"
        sidecar.istio.io/userVolume: '[{"name":"wasmfilters-dir","emptyDir": {}}]'
        sidecar.istio.io/userVolumeMount: '[{"mountPath":"/var/local/lib/wasm-filters","name":"wasmfilters-dir"}]'
      labels:
        app: hello-node
    spec:
      containers:
        - name: node-hello-world
          image: quay.io/jcarvaja/node-hello-world
          imagePullPolicy: Always
      restartPolicy: Always
EOF
```

Restart the pods, so the new folder is created in the istio-proxy container:

```sh
> kubectl delete pod -l app=hello-node -n my-app
```

Then, copy our oidc filter to the istio proxy container:

```sh
> PODS_TO_UPDATE=$(kubectl get pods -lapp=hello-node -n my-app -o jsonpath='{.items[0].metadata.name}')
> kubectl cp ./oidc.wasm ${PODS_TO_UPDATE}:/var/local/lib/wasm-filters/oidc.wasm -n my-app --container istio-proxy
```

And finally, we need to update the envoy filter to use the oidc filter using the Keycloak configuration we setup in section 2:

```sh
> kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: oidc-filter
  namespace: my-app
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      proxy:
        proxyVersion: '^1\.6.*'
      listener:
        filterChain:
          filter:
            name: envoy.http_connection_manager
            subFilter:
              name: envoy.filters.http.jwt_authn
    patch:
      operation: INSERT_BEFORE
      value:
        config:
          config:
            name: oidc-filter
            rootId: oidc-filter_root
            configuration: |
                {
                  "auth_cluster": "outbound|8080||keycloak.default.svc.cluster.local",
                  "auth_host": "keycloak.default.svc.cluster.local:8080",
                  "login_uri": "http://localhost:8080/auth/realms/master/protocol/openid-connect/auth",
                  "token_uri": "http://localhost:8080/auth/realms/master/protocol/openid-connect/token",
                  "client_id": "test", 
                  "client_secret": "$CLIENT_SECRET"
                }
            vmConfig:
              code:
                local:
                  filename: /var/local/lib/wasm-filters/oidc.wasm
              runtime: envoy.wasm.runtime.v8
              vmId: oidc-filter
              allow_precompiled: true
        name: envoy.filters.http.wasm
  workloadSelector:
    labels:
      app: hello-node
EOF
```

| The client ID must be **test** as it's hardcoded in [the oidc filter source](https://github.com/dgn/oidc-filter/blob/master/src/lib.rs#L92). 

## Conclusions

We have introduced how to secure our apps deployed in K8s using Istio. Also, we have configured the redirect mecanism in our apps with zero changes in our services. 
Also, we have used Minikube to have a K8S local environment, however we could have used [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/) too.

Credits to [this tech talk](https://developers.redhat.com/devnation/tech-talks/istio-webassembly) from RedHat.
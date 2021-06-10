---
layout: post
title: Deploy Quarkus Apps into the Cloud
date: 2021-06-10
tags: [ Quarkus ]
---

As part of my position on the QE team in [Quarkus](https://quarkus.io/), we need to play with a lot of integration tools and frameworks. Also, we need to use many cloud infrastructures like OpenShift and Kubernetes. 

This post is a guide to deploy Quarkus applications into cloud infrastructures. For now, it only covers OpenShift and Kubernetes, but I will try to add all the options that Quarkus allows like AWS and Azure in the future. Moreover, I will try to keep these steps up-to-date.

## Getting Started

Before going forward, I'm assuming that you already have a Quarkus application in place. If not, please proceed first with [the official getting started guide](https://quarkus.io/get-started/) in Quarkus.

As an example, using the [Quarkus CLI](https://quarkus.io/version/main/guides/cli-tooling) tool, you can easily create a Quarkus application by doing:

```
quarkus create app
```

This command will create a simple Quarkus application at `./code-with-quarkus`. 

## OpenShift

For all the methods, we recommend first to create a new OpenShift project where to deploy our resources:

```
oc new-project my-project
```

### Using s2i: From Binaries and Sources

OpenShift offers a really useful feature called [Source-to-IMage (s2i)](https://docs.openshift.com/container-platform/4.7/cicd/builds/understanding-image-builds.html#understanding-image-builds). Basically, OpenShift can build our App images from binaries or from a source repository. Let's see how to do this using a Quarkus application.

To use this method, we need to create our Build Configuration that will be used to generate our Quarkus applications. OpenShift s2i delegates the build onto a image, so the important bit is to select the right image.

For Quarkus, the right images are:

| Quarkus JVM  | Quarkus Native |
| ------------- | ------------- |
| registry.access.redhat.com/ubi8/openjdk-11 | quay.io/quarkus/ubi-quarkus-native-binary-s2i:1.0 |

In this tutorial, we're using only JVM build, so I will be using only `registry.access.redhat.com/ubi8/openjdk-11`.

- **From Binaries**

1. Create the Build configuration

```
oc new-build registry.access.redhat.com/ubi8/openjdk-11 --name quarkus-app --binary=true
```

Or simply apply the YAML format:

```yaml
apiVersion: "build.openshift.io/v1"
kind: "BuildConfig"
metadata:
  name: "quarkus-app"
spec:
  source:
    binary: {}
  strategy:
    sourceStrategy:
      from:
        kind: "DockerImage"
        name: "registry.access.redhat.com/ubi8/openjdk-11"
```

Now, let's trigger the previous Build configuration using our binaries.

2. Then, build your Quarkus application

```bash
mvn clean install
```

3. Trigger the s2i from the `target/quarkus-app` location

| Note that we're assuming that you're using [the `fast-jar` package type](https://quarkus.io/guides/maven-tooling#fast-jar) which is the default option. If not, basically you need to pick the folder where the `.jar` file is.

```
oc start-build quarkus-app --from-dir=target --follow=true --wait
```

When finished, the App image should be installed in our project:

```
> oc get imagestream
NAME          IMAGE REPOSITORY                                                                 TAGS     UPDATED
quarkus-app   default-route-openshift-image-registry.apps.XXXX/my-project/quarkus-app          latest   41 minutes ago
```

4. Let's deploy it!

```
oc new-app quarkus-app:latest
```

And expose it:

```
oc expose svc/quarkus-app
```

Check out the generated routes for our app:

```
> oc get routes
NAME           HOST/PORT                                 PATH   SERVICES   PORT       TERMINATION   WILDCARD
quarkus-app    quarkus-app-my-project.apps.XXXX          app               8080-tcp                 None
```

And we should be able to invoke our App which is up and running in our OCP instance!

```
> curl quarkus-app-my-project.apps.XXXX/hello
Hello RESTEasy!
```

- **From Sources**

This method is even easiest, but we need our sources to be publicly available in a repository like [github.com](github.com) and then create all the application resources in one go:

```
oc new-app registry.access.redhat.com/ubi8/openjdk-11~https://github.com/quarkusio/quarkus-quickstarts.git --context-dir=getting-started --name=quarkus-app
```

| Note that `registry.access.redhat.com/ubi8/openjdk-11` is the image builder, see the introduction for further details.

This command will create the deployment config, the service, the build config and will start the build in one go. In order to see how the build goes:

```
oc logs -f bc/quarkus-app
```

When finished, we can expose the service and use it:

```
oc expose svc/quarkus-app
```

Check out the generated routes for our app:

```
> oc get routes
NAME           HOST/PORT                                 PATH   SERVICES   PORT       TERMINATION   WILDCARD
quarkus-app    quarkus-app-my-project.apps.XXXX          app               8080-tcp                 None
```

And we should be able to invoke our App which is up and running in our OCP instance!

```
> curl quarkus-app-my-project.apps.XXXX/hello/greeting/quarkus
hello quarkus
```

### Using Quarkus OpenShift extension

Is there an even easiest way to deploy Quarkus apps into OpenShift? Yes! Using [the Quarkus OpenShift](https://quarkus.io/guides/deploying-to-openshift) extension. 

All we need is to add the Quarkus OpenShift extension into our pom.xml:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-openshift</artifactId>
</dependency>
```

Or using the [Quarkus CLI](https://quarkus.io/version/main/guides/cli-tooling) tool:

```
> quarkus extension add quarkus-openshift
[SUCCESS] ✅ Extension io.quarkus:quarkus-openshift has been installed
```

And deploy it using the Maven command:

```
mvn clean package -Dquarkus.kubernetes.deploy=true -Dquarkus.openshift.route.expose=true
```

| Note that if you are using an OpenShift with unsecure SSL, you also need to append `-Dquarkus.kubernetes-client.trust-certs=true` to the Maven command.

| Internally, this method will use the Binaries s2i to build the image and then start the application. However, you can configure this method by using either Docker or [jib](https://github.com/GoogleContainerTools/jib). More in [here](https://quarkus.io/version/main/guides/deploying-to-openshift#non-s2i-builds).

When the Maven command finished, your app should be up and running!

```
> oc get routes
NAME           HOST/PORT                                 PATH   SERVICES   PORT       TERMINATION   WILDCARD
quarkus-app    quarkus-app-my-project.apps.XXXX          app               8080-tcp                 None
```

And we should be able to invoke our App which is up and running in our OCP instance!

```
> curl quarkus-app-my-project.apps.XXXX/hello
Hello RESTEasy!
```

### Manual: Using Container Registry

Do you like to do everything on your own? Fair enough! Let's explain how to deploy Quarkus applications without using s2i or the Quarkus OpenShift extension, but directly docker and a container registry where to push/pull images.

First thing you need is to have access to a container registry and be logged in:

```
docker login <container-registry>
```

Where `<container-registry>` could be `quay.io`.

Then, we need to build our images locally. The good news is that the auto generated project by code starters contains the necessary `Dockerfile`, so all you need to do is:

```
> docker build -f src/main/docker/Dockerfile.jvm -t quarkus-app:latest .
(...)
Successfully tagged quarkus-app:latest
```

| This example only shows the JVM build, but the native Dockerfile is also present by default when generating a new Quarkus project.

Let's push the recently created `quarkus-app:latest` image into our container registry:

```
docker tag quarkus-app:latest <container-registry>/<namespace>/quarkus-app:latest
docker push <container-registry>/<namespace>/quarkus-app:latest
```

Now, we can create our application in our OpenShift project using this image:

```
oc new-app <container-registry>/<namespace>/quarkus-app:latest
```

And expose it:

```
oc expose svc/quarkus-app
```

Check out the generated routes for our app:

```
> oc get routes
NAME           HOST/PORT                                 PATH   SERVICES   PORT       TERMINATION   WILDCARD
quarkus-app    quarkus-app-my-project.apps.XXXX          app               8080-tcp                 None
```

And we should be able to invoke our App which is up and running in our OCP instance!

```
> curl quarkus-app-my-project.apps.XXXX/hello
Hello RESTEasy!
```

## Kubernetes

We don't have s2i in Kubernetes, but there are other many options to deploy applications into Kubernetes. For now, I've only included the container registry strategy.

Again, we recommend first to create a new Kubernetes project where to deploy our resources:

```
kubect create namespace my-project
```

### Manual: Using Container Registry

First thing you need is to have access to a container registry and be logged in:

```
docker login <container-registry>
```

Where `<container-registry>` could be `quay.io`.

Then, we need to build our images locally. The good news is that the auto generated project by code starters contains the necessary `Dockerfile`, so all you need to do is:

```
> docker build -f src/main/docker/Dockerfile.jvm -t quarkus-app:latest .
(...)
Successfully tagged quarkus-app:latest
```

| This example only shows the JVM build, but the native Dockerfile is also present by default when generating a new Quarkus project.

Let's push the recently created `quarkus-app:latest` image into our container registry:

```
docker tag quarkus-app:latest <container-registry>/<namespace>/quarkus-app:latest
docker push <container-registry>/<namespace>/quarkus-app:latest
```

Now, we can create our application in our Kubernetes project using this image:

```
kubectl create deployment --image=<container-registry>/<namespace>/quarkus-app:latest quarkus-app
```

And expose it:

```
kubectl expose deployment quarkus-app --port=8080 --name=quarkus-app-http
```

And we should be able to invoke our App which is up and running in our Kubernetes instance!

## Future Cloud Infrastructure to cover

- Amazon
- Azure
- Google Cloud
---
layout: post
title: Getting started with Java Operator SDK
date: 2022-02-10
tags: [ Java, Operator, Kubernetes ]
---

[Operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) are a super useful tool in Kubernetes to manage and handle custom resources. But what is a custom resource? Anything! For example, a custom resource could be an object that contains information about how to connect to an API:

```yaml
apiVersion: my.group/v1
kind: Connection
metadata:
  name: my-connection-config
spec:
  url: http://xxx
  user: abc
  pass: secret
```

Let's now imagine that we want to (1) verify the `url` field within the `my-connection-config` custom resource is valid and then (2) transform this custom resource to a [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) so it can be used by other applications. Congrats! This kind of things are what the operators are meant to!

So, I need to implement an operator that verify my custom resource and transform it to a ConfigMap, but how can implement it? Kubernetes has [an operator SDK](https://github.com/operator-framework/operator-sdk) to implement operators in Go and there are thousands of guides about how to do it. However, spite of Go is easy to learn, generally writing operators can become a tedious work. The good news is that a new [Java Operator SDK](https://javaoperatorsdk.io/) has been published, not only to write operators in Java, but also for easing the development of operators.

In this post, I will explore out the Java Operator SDK and implement a very basic operator to traslate a text from `Hello World` to `Hola Mundo`.

| Note that the Java Operator SDK is also available as a Quarkus extension: [https://github.com/quarkiverse/quarkus-operator-sdk](https://github.com/quarkiverse/quarkus-operator-sdk).

## Requirements
- [Maven](https://maven.apache.org/)
- [Java 11](https://openjdk.java.net/)

## Getting Started

Let's create our Maven project first:

```
mvn archetype:generate -DgroupId=org.sgitario.operator -DartifactId=translator -DinteractiveMode=false
```

And import the java operator sdk pom as dependency management:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.javaoperatorsdk</groupId>
      <artifactId>java-operator-sdk</artifactId>
      <!-- I used 2.1.1 for this post -->
      <version>{see https://search.maven.org/search?q=a:operator-framework%20AND%20g:io.javaoperatorsdk for latest version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

Next, let's configure our project using Java 11:

```xml
<properties>
  <maven.compiler.source>11</maven.compiler.source>
  <maven.compiler.target>11</maven.compiler.target>
</properties>
```

Let's add the java operator framework dependency:

```xml
<dependencies>
  <dependency>
    <groupId>io.javaoperatorsdk</groupId>
    <artifactId>operator-framework</artifactId>
  </dependency>
</dependencies>
```

And the Fabric8 dependency to autogenerate the resources for the custom resources:

```xml
<dependency>
  <groupId>io.fabric8</groupId>
  <artifactId>crd-generator-apt</artifactId>
  <!-- This is provided because it's only need at build time -->
  <scope>provided</scope>
</dependency>
```

Now, let's create our custom resource at `src/main/java`:

```java
package org.sgitario.operator;

import io.fabric8.kubernetes.api.model.Namespaced;
import io.fabric8.kubernetes.client.CustomResource;
import io.fabric8.kubernetes.model.annotation.Group;
import io.fabric8.kubernetes.model.annotation.Version;

@Group("operator.translator.text")
@Version("v1")
public class Text extends CustomResource<TextSpec, TextStatus> implements Namespaced {
}
```

In Kubernetes, the custom resources have generally three parts:
- `metadata` - for names, annotations, etc
- `spec` - the actual content of the custom resource
- `status` - to be updated by the cluster... and operators!

In the Java Operator SDK, we simply need to extend our custom resource with `CustomResource` and provide the spec `TextSpec` and the status `TextStatus` implementations:

```java
public class TextSpec {

  private String text;

  public String getText() {
    return text;
  }

  public void setText(String text) {
    this.text = text;
  }
}

public class TextStatus extends ObservedGenerationAwareStatus {

  private String translated;

  public String getTranslated() {
    return translated;
  }

  public void setTranslated(String translated) {
    this.translated = translated;
  }
}
```

We can see the `spec` as the input for our operator and the output is the `status` part. Let's implement our controller operator that simply replaces the `Hello world!` text to `Hola mundo!`:

```java
package org.sgitario.operator;

import io.javaoperatorsdk.operator.api.reconciler.Context;
import io.javaoperatorsdk.operator.api.reconciler.ControllerConfiguration;
import io.javaoperatorsdk.operator.api.reconciler.Reconciler;
import io.javaoperatorsdk.operator.api.reconciler.UpdateControl;

@ControllerConfiguration
public class TextReconciler implements Reconciler<Text> {
    public UpdateControl<Text> reconcile(Text crd, Context context) {
        String text = crd.getSpec().getText();
        crd.setStatus(new TextStatus());
        crd.getStatus().setTranslated(text.replace("Hello world!", "Hola mundo!"));
        return UpdateControl.updateStatus(crd);
    }
}
```

Our controller will receive our custom resource `Text` as an event and our job is to reconciliate the custom resource and its status. In other words, we need to update the status and mark it as done using `UpdateControl.updateStatus`. You can find more information about controllers and reconciliation in [here](https://cluster-api.sigs.k8s.io/developer/providers/implementers-guide/controllers_and_reconciliation.html).

So far, we have added our custom resource `Text` and the controller of our operator `TextReconciler`, but we didn't add the actual operator application:

```java
package org.sgitario.operator;

import io.javaoperatorsdk.operator.Operator;
import io.javaoperatorsdk.operator.config.runtime.DefaultConfigurationService;

public class WebPageOperator {
    public static void main(String[] args) {
        Operator operator = new Operator(DefaultConfigurationService.instance());
        operator.register(new WebPageReconciler());
        operator.start();
    }
}
```

Before deploying our operator, let's test it first.

## Testing

The Java Operator SDK provides a JUnit 5 library to ease the testing of our operators, so let's add it to our `pom.xml` file:

```xml
<dependency>
  <groupId>io.javaoperatorsdk</groupId>
  <artifactId>operator-framework-junit-5</artifactId>
  <!-- I used 2.1.1 for this post -->
  <version>{see https://search.maven.org/search?q=a:operator-framework-junit-5%20AND%20g:io.javaoperatorsdk for latest version}</version>
  <scope>test</scope>
</dependency>
```

| Unfortunately, this dependency is not part of the pom `java-operator-sdk:java-operator-sdk`, so we need to set the version.

Make sure you're using at least the version `2.22.2` of surefire, otherwise the JUnit 5 tests won't run:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>2.22.2</version>
    </plugin>
  </plugins>
</build>
```

At the moment, we can't use mock servers like [fabric8:kubernetes-server-mock](https://mvnrepository.com/artifact/io.fabric8/kubernetes-server-mock), so before running our tests, we need a Kubernetes cluster up and running. For writing this post, I used [Kind](https://kind.sigs.k8s.io/) and it worked pretty well. But you can also use [Minikube](https://minikube.sigs.k8s.io/) or [K3s](https://k3s.io/).

Assuming you have already the Kubernetes cluster up and running, let's continue writing the test at `src/test/java`:

```java
package org.sgitario.operator;

import static java.util.concurrent.TimeUnit.MINUTES;
import static org.awaitility.Awaitility.await;
import static org.hamcrest.CoreMatchers.is;
import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.notNullValue;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import io.fabric8.kubernetes.api.model.ObjectMetaBuilder;
import io.javaoperatorsdk.operator.config.runtime.DefaultConfigurationService;
import io.javaoperatorsdk.operator.junit.AbstractOperatorExtension;
import io.javaoperatorsdk.operator.junit.OperatorExtension;

public class TextOperatorTest {

    @RegisterExtension
    AbstractOperatorExtension operator = OperatorExtension.builder()
            .withConfigurationService(DefaultConfigurationService.instance())
            .withReconciler(new TextReconciler())
            .build();

    @Test
    public void test() {
        // Given input custom resource called "mycrd"
        Text crd = new Text();
        crd.setMetadata(new ObjectMetaBuilder()
                .withName("mycrd")
                .withNamespace(operator.getNamespace())
                .build());
        crd.setSpec(new TextSpec());
        crd.getSpec().setText("Hello world!");

        // When we create it in the cluster 
        operator.getKubernetesClient().resource(crd).createOrReplace();

        // Then the operator should be triggered and should update the "translated" field in the status
        // Waiting 1 minute for expected resources to be created and updated
        await().atMost(1, MINUTES).ignoreExceptions().untilAsserted(() -> {
            Text updatedSchema =
                    operator.getKubernetesClient().resources(Text.class).inNamespace(operator.getNamespace())
                            .withName(crd.getMetadata().getName()).get();
            assertThat(updatedSchema.getStatus(), is(notNullValue()));
            assertThat(updatedSchema.getStatus().getTranslated(), is("Hola mundo!"));
        });
    }
}
```

The test framework will create a temporal namespace before starting the test where will load all the generated resources at `target/classes/META-INF/fabric8` and will destroy the temporal namespace after the test execution. The resources at `target/classes/META-INF/fabric8` include the custom resource definitions that are generated by fabric8 when adding the Maven dependency `io.fabric8:crd-generator-apt`.
Also, take into account that the operator is not triggered immediately, so we need to wait some time for the resource to be updated.

So good so far, we have our operator tested and ready to be deployed, let's do this!

## Deployment

First of all, we will deploy our operator in the namespace `translator-operator`:

```
kubectl create namespace translator-operator
```

Next, we need to generate a container image of our operator. For doing this, we'll use the jib Maven plugin:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>com.google.cloud.tools</groupId>
      <artifactId>jib-maven-plugin</artifactId>
      <version>3.2.0</version>
      <configuration>
        <from>
          <image>gcr.io/distroless/java:11</image>
        </from>
        <to>
          <image>quay.io/jcarvaja/translator-operator</image> <!-- change this property to your image -->
        </to>
      </configuration>
    </plugin>
  </plugins>
</build>
```

And run it:

```
mvn jib:dockerBuild
docker push quay.io/jcarvaja/translator-operator
```

We have our image built and pushed in the container registry. Now, we need to load the manifests into the cluster.

The right approach to deploy operators and be available for users is via [the Operator Lifecycle Manager](https://olm.operatorframework.io/) which is really well integrated with [the Operator SDK](https://sdk.operatorframework.io/docs/olm-integration/). I will dig into this approach more in future updates.

Basically, what we will need are the following manifests:
- **CRDs** that define the APIs your operator will manage.
- **Operator** resource containing the Deployment that runs your operator pods.
- **RBAC** (usually the role.yaml, role_binding.yaml, service_account.yaml files) that configures the service account permissions your operator requires.

The **CRDs** manifests have been autogenerated by Fabric8 at `target/classes/META-INF/fabric8` already, so let's load them:

```
kubectl apply -f target/classes/META-INF/fabric8/texts.sample.javaoperatorsdk-v1.yml -n translator-operator
```

Let's see how looks like the **Operator** manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: translator-operator
spec:
  selector:
    matchLabels:
      app: translator-operator
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: translator-operator
    spec:
      serviceAccountName: translator-operator
      containers:
      - name: operator
        image: quay.io/jcarvaja/translator-operator # We need to push our operator image here!
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

```
kubectl apply -f operator.yml -n translator-operator
```

Let's now add the rbac resources:

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: translator-operator
  namespace: translator-operator
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: translator-operator
  namespace: translator-operator
rules:
- apiGroups:
  - sample.javaoperatorsdk
  resources:
  - texts
  verbs:
  - "*"
- apiGroups:
  - sample.javaoperatorsdk
  resources:
  - texts/status
  verbs:
  - "*"
- apiGroups:
  - apiextensions.k8s.io
  resources:
  - customresourcedefinitions
  verbs:
  - "get"
  - "list"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: translator-operator
  namespace: translator-operator
subjects:
- kind: ServiceAccount
  name: translator-operator
  namespace: translator-operator
roleRef:
  kind: ClusterRole
  name: translator-operator
  apiGroup: ""
```

```
kubectl apply -f rbac.yml
```

And now, our operator should be up and running:

```
kubectl get pods -n translator-operator
NAME                                  READY   STATUS    RESTARTS   AGE
translator-operator-d9b8cf899-hj5fm   1/1     Running   0          6m
```

Let's create a custom resource and verify the operator it's working fine now:

```yaml
apiVersion: "sample.javaoperatorsdk/v1"
kind: Text
metadata:
  name: my-text
spec:
  text: Hello world!
```

```
kubectl apply -f resource.yml -n translator-operator
```

When adding it, our operator you add a new `translated` field in the status:
```
kubectl describe texts/my-text -n translator-operator
```

Output:
```yaml
...
spec:
  text: Hello world!
status:
  observedGeneration: 1
  translated: Hola mundo!
```

T'chan!! It worked!

## Conclusions

- Missing a Maven archetype with a basic example with fabric8 and jib configured.

I think it would be useful to have a simple command to generate a skeleton: `mvn archetype:generate -DgroupId=org.sgitario.operator -DartifactId=translator -DinteractiveMode=false -DarchetypeGroupId=io.javaoperatorsdk -DarchetypeArtifactId=maven-archetype-webpage`.

This skeleton should include a basic example with tests and all plugins.

| For the Quarkus extension of the Java Operator SDK, we can use: `operator-sdk init --plugins quarkus --domain halkyon.io --project-name expose`. Operator SDK is a command tool that can be installed from [here](https://sdk.operatorframework.io/docs/installation/).

- The documentation mentions the Fabric8 dependency `crd-generator-apt` as optional and from my point of view, it should be a must. Also, when running the tests, it won't work if this dependency is not configured.

- We can't use mocks of Kubernetes clients in tests

This is because the JUnit5 extension creates an instance of the Kubernetes client and does not allow to use the mocked one. See [the line](https://github.com/java-operator-sdk/java-operator-sdk/blob/dee38193d27fb2c43028bc85172ff6a2cf0e0e68/operator-framework-junit5/src/main/java/io/javaoperatorsdk/operator/junit/AbstractOperatorExtension.java#L37).

- The pom `io.javaoperatorsdk:java-operator-sdk` does not include neither the `operator-framework-junit-5` dependency nor the plugin `jib-maven-plugin`. 

This is mainly to avoid having to set the version.

- At the beginning, a runtime exception was being thrown when running the tests. However, nothing was printed in the console which made difficult to troubleshot the issue. 

- Even though, I only use the version `v1` in my custom resource, the generated resources create two versions `v1` and `v1beta1`. 

---
layout: post
title: Deploy Quarkus Applications with Databases in Kubernetes
date: 2023-02-06
tags: [ Containers, Quarkus ]
---

Quarkus is a super-performant and cloud-native framework. Nowadays, I'm part of the Quarkus development team and I've written a lot of posts here about Quarkus, so let's go directly to the topic!

Spites there are a lot of posts that explain how to build and deploy Quarkus applications in Kubernetes, there seems to be a lack of information about how to achieve this when our Quarkus applications depend on other services like databases. Let's address this in this post.

# Prerequisities

- Maven 3.5+
- Java 11+
- Have connected to a Kubernetes cluster using [Kubectl](https://kubernetes.io/docs/reference/kubectl/)
- Have logged in to a Container registry like Quay.io or Docker Hub. 

# Index

- [Create Quarkus Application](#Create-Quarkus-Application)
- [Add Kubernetes extension](#Add-Kubernetes-extension)
- [Configure Container Image extension](#Configure-Container-Image-extension)
- [(Optional) Start the database when installing our application](#optional-start-the-database-when-installing-our-application)
- [Add the configMap with the datasource configuration](#add-the-configmap-with-the-datasource-configuration)

# Create Quarkus Application

Our use case is really simple and easy to understand: we're going to implement a Quarkus application that stores and retrieves fruits from a database. For doing this, we're going to use the following Quarkus extensions:

- [Resteasy Reactive Jackson](https://quarkus.io/guides/resteasy-reactive): REST + JSON support
- [Hibernate ORM REST Data with Panache](https://quarkus.io/guides/rest-data-panache): Database support + Automatically generate REST from entities

First of all, we're going to create the Quarkus application from zero using the Quarkus Maven plugin:

```s
mvn io.quarkus.platform:quarkus-maven-plugin:2.16.1.Final:create -Dextensions=resteasy-reactive-jackson,quarkus-hibernate-orm-rest-data-panache
```

After providing your group ID and the artefact name, enter to the generated folder.

As you selected the `resteasy-reactive-jackson` extension, the Hello resource is automatically created. So, if we run the application in DEV mode:

```s
mvn quarkus:dev
```

And you call to the hello resource, you should see `Hello from RESTEasy Reactive`:

```s
curl localhost:8080/hello
> Hello from RESTEasy Reactive
```

We're now going to generate our first entity `Fruit` with only one column `name` (plus the `id` primary key column coming from `PanacheEntity`):

```java
import javax.persistence.Entity;

import io.quarkus.hibernate.orm.panache.PanacheEntity;

@Entity
public class Fruit extends PanacheEntity {
    public String name;
}
```

And we're also going to expose the Fruit entity via REST endpoints. We will use the REST Data with Panache extension to do this.

```java
import io.quarkus.hibernate.orm.rest.data.panache.PanacheEntityResource;

public interface FruitResource extends PanacheEntityResource<Fruit, Long> {
}
```

The interface `PanacheEntityResource` needs the entity and the primary key column which is a Long (see `PanacheEntity` implementation).

Additionaly, let's add some initial data by creating the SQL script `import.sql` in `src/main/resources`:

```sql
insert into Fruit(id, name) values (1, 'apple');
insert into Fruit(id, name) values (2, 'banana');
```

Now, if we run again our project using Quarkus dev mode, we'll see this exception:

```
ERROR [io.qua.dep.dev.IsolatedDevModeMain] (main) Failed to start quarkus: java.lang.RuntimeException: io.quarkus.builder.BuildException: Build failure: Build failed due to errors
  [error]: Build step io.quarkus.hibernate.orm.deployment.HibernateOrmProcessor#configurationDescriptorBuilding threw an exception: io.quarkus.runtime.configuration.ConfigurationException: Model classes are defined for the default persistence unit, but no default datasource was found. The default EntityManagerFactory will not be created. To solve this, configure the default datasource. Refer to https://quarkus.io/guides/datasource for guidance.
```

This is because we have not configured any JDBC driver yet. But Quarkus helps us by saying that we should go to (https://quarkus.io/guides/datasource)[https://quarkus.io/guides/datasource] for further information. In this guide, we'll see [here](https://quarkus.io/guides/datasource#jdbc-datasource-2) that we have a lot of drivers available. Let's use the `jdbc-postgresql` POSTGRESQL driver:

```s
mvn quarkus:add-extension -Dextensions='jdbc-postgresql'
```

If we run again our project in DEV mode, we'll see that our Fruit resource is generated with the initial data:

```s
curl localhost:8080/fruit
> [{"id":1,"name":"apple"},{"id":2,"name":"banana"}]
```

It magically worked! Unfortunately, this only works in DEV mode :( Since you selected the JDBC postgresql driver, the DEV mode is automatically starting a postgresql instance using docker in your local machine and it's also configuring your Quarkus application to use this Postgresql instance. 

So, when running the application directly using the JAR file: 

```s
# Let's first build the app
mvn clean install
# And now, let's run it
java -jar target/quarkus-app/quarkus-run.jar 
# it will fail! :(
> Model classes are defined for the default persistence unit <default> but configured datasource <default> not found: the default EntityManagerFactory will not be created. To solve this, configure the default datasource. Refer to https://quarkus.io/guides/datasource for guidance.
```

Therefore, you will eventually need to deal with how to setup Quarkus when running in Kubernetes. Let's see how to do this in the following sections!

Note that, for demo purposes, we will also add the following two properties to import the initial data from the import.sql file and to drop/create the schema everytime our application starts:

```
quarkus.hibernate-orm.sql-load-script=import.sql
quarkus.hibernate-orm.database.generation=drop-and-create
```

# Mapping Data Source properties

What DEV mode is doing internally, is to configure the following three properties after starting the Postgresql instance: 

- `quarkus.datasource.jdbc.url`
- `quarkus.datasource.username`
- `quarkus.datasource.password`

So, we need to somehow do the same when running our application in Kubernetes. The problem is that we might not know what the Postgresql JDBC URL, or username or password values are when building our application. 

To allow us to provide these properties when installing the application instead of when building the application, we can map these properties to environmental properties:

```
quarkus.datasource.jdbc.url=${POSTGRESQL_URL}
quarkus.datasource.username=${POSTGRESQL_USERNAME}
quarkus.datasource.password=${POSTGRESQL_PASSWORD}
```

If we run our application now, the application will still fail to start because these environmental properties do not exist yet which is expected. Let's see how we can provide these environmental properties in Kubernetes.

# Add Kubernetes extension

Beforehand, to deploy applications in Kubernetes, we need to write the Kubernetes resources like the Service, Ingress, Deployment resources. Luckily, Quarkus has [the Kubernetes extension](https://quarkus.io/guides/deploying-to-kubernetes) that will generate all these resources when building your application:

```s
mvn quarkus:add-extension -Dextensions='kubernetes'
mvn clean install
```

After building the application, you will find the generated Kubernetes resources at `target/kubernetes/kubernetes.yml`.

But, as we saw in the previous section, we need to provide several environmental properties. We're going to provide these properties using a ConfigMap resource that we'll create and install before installing our application. To instruct our Quarkus application to read the environment properties from a ConfigMap resource, we need to add the following properties:

```
quarkus.kubernetes.env.configmaps=postgresql-datasource-props
```

After building again our project, we'll see that the generated Deployment resource in `target/kubernetes/kubernetes.yml` will contain the following container:

```yaml
containers:
  - envFrom:
      - configMapRef:
          name: postgresql-datasource-props
```

# Configure Container Image extension

We have built the Kubernetes resources to install our application, but we also need to build and push the container image of our application, so Kubernetes can pull this image to start our application. And again, Quarkus provides an easy way to do this by selecting one of the several [container image](https://quarkus.io/guides/container-image) extensions. Here, we'll use [the Container Image Docker](https://quarkus.io/guides/container-image#docker) extension:

```s
mvn quarkus:add-extension -Dextensions='container-image-docker'
```

And let's configure the final image that will be built using the following property:

```
quarkus.container.image.image=<REGISTRY>/<GROUP>/<NAME>:<TAG>
# example: quarkus.container.image.image=quay.io/user/app:1.0
```

And let's push the built image into the registry using the property `quarkus.container-image.push=true`.

```s
mvn clean install -Dquarkus.container-image.push=true
```

Now, the container image should be published in our container registry. 

# (Optional) Start the database when installing our application

There are many, many ways to install databases in Kubernetes: either via a database operator, or Helm, ... 
If you're already have a database instance ready to use, you can skip this step. Otherwise, let's see how you can include the database deployment as part of the Quarkus application installation in Kubernetes.

We're going to use the most basic setup to start a Postgresql instance. First, you need to create the file `kubernetes.yml` (or `common.yml`) in `src/main/kubernetes` folder. And then, add the following Deployment resource in this file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      service: postgres
  template:
    metadata:
      labels:
        service: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15.1
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: example
            - name: POSTGRES_USER
              value: user
            - name: POSTGRES_PASSWORD
              value: pass
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  type: ClusterIP
  ports:
    - port: 5432
  selector:
    service: postgres
```

As you can see, we have provided the database name `example`, the user `user` and the password `pass`. So, we need to bind these values to the environment properties we have configured in [this section](#Mapping-Data-Source-properties).

# Add the configMap with the datasource configuration

Let's provide the Postgresql configuration using a ConfigMap resource named "postgresql-datasource-props". 

We're going to add this ConfigMap resource in `src/main/kubernetes/kubernetes.yml`, so it's aggregated to the generated `target/kubernetes/kubernetes.yml` and hence we can deploy everything in one go. But you can also create this ConfigMap resource later right before of installing the application.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgresql-datasource-props
data:
  POSTGRESQL_URL: "jdbc:postgresql://postgres:5432/example"
  POSTGRESQL_USERNAME: "user"
  POSTGRESQL_PASSWORD: "pass"
```

After building the project, you should see this ConfigMap resource within the `target/kubernetes/kubernetes.yml`.

# Deploy the Quarkus application with database

Finally, let's deploy our Quarkus application and also the database in one single step by doing:

Note that you should have been connected to Kubernetes and have selected a namespace where to deploy the application!

```
mvn clean install -Dquarkus.container-image.push=true -Dquarkus.kubernetes.deploy=true
```

# Conclusion

We have seen how easy is to use Quarkus to deploy applications that need databases in Kubernetes. In future posts, I'm going to introduce [the Quarkus Helm](https://github.com/quarkiverse/quarkus-helm) extension that make things much easier. 
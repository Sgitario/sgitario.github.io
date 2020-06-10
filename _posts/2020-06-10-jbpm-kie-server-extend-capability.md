---
layout: post
title: Extend Capability in Kie Server
date: 2020-06-10
tags: [ jbpm ]
---

The Kie Server REST API enables you to interact with our business assets (such as business rules, processes, and solvers). The available REST endpoints are determined by the **capabilities** enabled in our Kie Server. We can extend an existing Kie Server capability with a custom REST API endpoint to further adapt our business needs.

In this post, we're going to extend the BRM capability (Drools extension) using the Kie Server 7.8 image with the following custom REST API endpoint:

```
/server/containers/instances/{containerId}/customksession/{ksessionId}
```


1. Create Project

```
mvn archetype:generate -DgroupId=org.sgitario.jbpm -DartifactId=extend-capability -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

2. Add kie server dependencies into the **pom.xml**

Depending on the Kie Server image, you might need to change [the kie server dependencies](https://search.maven.org/artifact/org.kie/kie-api). For Kie Server 7.8 image, the dependencies version is **7.38.0.Final**.

```xml
<packaging>jar</packaging>

<properties>
  <version.org.kie>7.38.0.Final</version.org.kie>
</properties>

<dependencies>
  <dependency>
    <groupId>org.kie</groupId>
    <artifactId>kie-api</artifactId>
    <version>${version.org.kie}</version>
  </dependency>
  <dependency>
    <groupId>org.kie</groupId>
    <artifactId>kie-internal</artifactId>
    <version>${version.org.kie}</version>
  </dependency>
  <dependency>
    <groupId>org.kie.server</groupId>
    <artifactId>kie-server-api</artifactId>
    <version>${version.org.kie}</version>
  </dependency>
  <dependency>
    <groupId>org.kie.server</groupId>
    <artifactId>kie-server-services-common</artifactId>
    <version>${version.org.kie}</version>
  </dependency>
  <dependency>
    <groupId>org.kie.server</groupId>
    <artifactId>kie-server-services-drools</artifactId>
    <version>${version.org.kie}</version>
  </dependency>
  <dependency>
    <groupId>org.kie.server</groupId>
    <artifactId>kie-server-rest-common</artifactId>
    <version>${version.org.kie}</version>
  </dependency>
  <dependency>
    <groupId>org.drools</groupId>
    <artifactId>drools-core</artifactId>
    <version>${version.org.kie}</version>
  </dependency>
  <dependency>
    <groupId>org.drools</groupId>
    <artifactId>drools-compiler</artifactId>
    <version>${version.org.kie}</version>
  </dependency>
  <dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.25</version>
  </dependency>
</dependencies>
```

3. Implement **KieServerApplicationComponentsService** interface in order to filter out application resources depending on the caller extensions and transports

```java
public class CustomKieServerApplicationComponentsService implements KieServerApplicationComponentsService {

	private static final String OWNER_EXTENSION = "Drools";

	@Override
	public Collection<Object> getAppComponents(String extension, SupportedTransports type, Object... services) {
		// Do not accept calls from extensions other than the owner extension
		// Only REST transport is implemented
		if (!OWNER_EXTENSION.equals(extension) || !SupportedTransports.REST.equals(type)) {
			return Collections.emptyList();
		}

		RulesExecutionService rulesExecutionService = findByType(services, RulesExecutionService.class);
		KieServerRegistry context = findByType(services, KieServerRegistry.class);
		Object resource = new CustomKSessionResource(rulesExecutionService, context);
		return Arrays.asList(resource);
	}

	@SuppressWarnings("unchecked")
	private <T> T findByType(Object[] services, Class<T> clazz) {
		return Stream.of(services).filter(svc -> clazz.isAssignableFrom(svc.getClass())).map(svc -> (T) svc).findFirst()
				.get();
	}

}
```

4. Make the service discoverable

To make the new endpoint discoverable, create a _src/main/resources/META-INF/services/org.kie.server.services.api.KieServerApplicationComponentsService_ file and add the fully qualified class name of the KieServerApplicationComponentsService implementation class within the file:

```
org.sgitario.jbpm.CustomKieServerApplicationComponentsService
```

5. Build the project

```
mvn clean package
```

It will generate the file **target/extend-capability-1.0-SNAPSHOT.jar**.

6. Create our own image of Kie Server

Create a file called _Dockerfile_:

```
FROM registry.redhat.io/rhdm-7/rhdm-kieserver-rhel8:7.8.0
COPY target/extend-capability-1.0-SNAPSHOT.jar /opt/eap/standalone/deployments/ROOT.war/WEB-INF/lib/
```

| Note that we need to sign up in [the Red Hat registry](https://access.redhat.com/RegistryAuthentication).

Then, build the image:

```
docker build . --tag quay.io/jcarvaja/rhdm-kieserver-rhel8-extended-capability
```

7. Prepare the KJAR example

We're going to create a pretty simple example in Drools (more about this topic in [this post](https://sgitario.github.io/drools-introduction/)). This example will only say hello to persons.

First, we need a Maven repository to be accessible to push the KJar module and our Kie Server:

```
docker run -d -p 8081:8081 --name nexus sonatype/nexus
```

The default credentials is _admin_/_admin123_.

Create the **pom.xml**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd" xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <modelVersion>4.0.0</modelVersion>
  <groupId>org.jbpm.test</groupId>
  <artifactId>say-hello-kjar-example</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>kjar</packaging>
  <name>say-hello-kjar-example</name>
  
  <build>
    <plugins>
      <plugin>
        <groupId>org.kie</groupId>
        <artifactId>kie-maven-plugin</artifactId>
        <version>7.38.0.Final</version>
        <extensions>true</extensions>
      </plugin>
    </plugins>
  </build>

  <distributionManagement>
   <snapshotRepository>
      <id>nexus-snapshots</id>
      <url>http://localhost:8081/nexus/content/repositories/snapshots</url>
   </snapshotRepository>
</distributionManagement>
</project>
```

Then, the **Person.java* domain model:

```java
package org.jbpm.test;

public class Person  implements java.io.Serializable {
    
    private Integer age;
    private String name;

    // constructors, getters and setters
}
```

The **say-hello-person.drl** rule:

```drl
import org.jbpm.test.Person;

rule SayHello
  when
	  $p : Person(age >= 21)
  then
	  System.out.println("Hello " + $p.getName());
end
```
 
And finally, we build and publish the artifact into our Nexus instance:

```
mvn --settings settings.xml clean install deploy
```

8. Test it:

We first run our Kie Server image:

```
docker run -it -p 8080:8080 --rm --env KIE_ADMIN_USER=admin --env KIE_ADMIN_PWD=admin --env MAVEN_MIRROR_URL=http://localhost:8081/nexus/content/groups/public/ quay.io/jcarvaja/rhdm-kieserver-rhel8-extended-capability
```

Then, we deploy our KJAR example:

```
curl -X PUT "http://admin:admin@localhost:8080/services/rest/server/containers/sayHello" -H "accept: application/json" -H "Content-Type: application/json" -d "{ \"container-id\" : \"sayHello\", \"release-id\" : { \"group-id\" : \"org.jbpm.test\", \"artifact-id\" : \"say-hello-kjar-example\", \"version\" : \"1.0-SNAPSHOT\" }}"
```

And run our example:

```
curl -X POST "http://admin:admin@localhost:8080/services/rest/server/containers/instances/sayHello/customksession/ksession" -H "Content-Type: application/json" -d '[{"org.jbpm.test.Person": {"name": "john","age": 25}}]'
```

Where *ksession* is the name in the **kmodule.xml** of our KJAR module.

## Conclusion

All the source code can be found in [this repository](https://github.com/Sgitario/kie-server-extend-capability).
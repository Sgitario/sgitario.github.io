---
layout: post
title: New Custom Capability in Kie Server
date: 2020-06-11
tags: [ jbpm ]
---

By default, Kie Server extensions are exposed through REST or JMS data transports. We can extend Kie Server to support a custom data transport adding a new capability.

As an example, this procedure adds a custom data transport to the Kie Server 7.7.1 image that uses the Drools extension and that is based on [Apache MINA](https://mina.apache.org/), an open-source Java network-application framework. The example custom MINA transport exchanges string-based data that relies on existing marshalling operations and supports only JSON format.

## 1.- Create Project

```
mvn archetype:generate -DgroupId=org.sgitario.jbpm -DartifactId=new-capability -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

## 2.- Add kie server dependencies into the **pom.xml**

Depending on the Kie Server image, you might need to change [the kie server dependencies](https://search.maven.org/artifact/org.kie/kie-api). For Kie Server 7.7.1 image, the dependencies version is **7.38.0.Final**.

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
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>org.kie</groupId>
    <artifactId>kie-internal</artifactId>
    <version>${version.org.kie}</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>org.kie.server</groupId>
    <artifactId>kie-server-api</artifactId>
    <version>${version.org.kie}</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>org.kie.server</groupId>
    <artifactId>kie-server-services-common</artifactId>
    <version>${version.org.kie}</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>org.kie.server</groupId>
    <artifactId>kie-server-services-drools</artifactId>
    <version>${version.org.kie}</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>org.drools</groupId>
    <artifactId>drools-core</artifactId>
    <version>${version.org.kie}</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>org.drools</groupId>
    <artifactId>drools-compiler</artifactId>
    <version>${version.org.kie}</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.25</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>org.apache.mina</groupId>
    <artifactId>mina-core</artifactId>
    <version>2.1.3</version>
  </dependency>
</dependencies>

<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-assembly-plugin</artifactId>
      <version>3.1.1</version>

      <configuration>
        <descriptorRefs>
          <descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
      </configuration>

      <executions>
        <execution>
          <id>make-assembly</id>
          <phase>package</phase>
          <goals>
            <goal>single</goal>
          </goals>
        </execution>
      </executions>

    </plugin>
  </plugins>
</build>
```

| Note that we need to create a fat jar, so the Mina dependency is added (as it's not included in Kie Server by default) and also we need to exclude the others (using scope provided). Another alternative is to include the Mina dependency in the Kie Server classpath manually.

## 3.- Create Text Based Handler for Mina Transport

The handler expects to receive something like "containerId|argument" in order to invoke the "containerId" with the specified "argument". The flow will end when it receives either "exit".

```java
public class TextBasedIoHandlerAdapter extends IoHandlerAdapter {

	private static final Logger LOG = LoggerFactory.getLogger(TextBasedIoHandlerAdapter.class);

	private final KieContainerCommandService batchCommandService;

	public TextBasedIoHandlerAdapter(KieContainerCommandService batchCommandService) {
		this.batchCommandService = batchCommandService;
	}

	@Override
	public void messageReceived(IoSession session, Object message) throws Exception {
		String completeMessage = message.toString();
		LOG.debug("Received message '{}'", completeMessage);
		if (completeMessage.trim().equalsIgnoreCase("exit")) {
			session.closeOnFlush();
			return;
		}

		String[] elements = completeMessage.split("\\|");
		LOG.debug("Container id {}", elements[0]);
		ServiceResponse<String> result = batchCommandService.callContainer(elements[0], elements[1],
				MarshallingFormat.JSON, null);

		if (result.getType().equals(ServiceResponse.ResponseType.SUCCESS)) {
			session.write(result.getResult());
			LOG.debug("Successful message written with content '{}'", result.getResult());
		} else {
			session.write(result.getMsg());
			LOG.debug("Failure message written with content '{}'", result.getMsg());
		}
	}
}
```

## 4.- Implement **KieServerExtension** interface in order to filter out application resources depending on the caller extensions and transports

```java
public class MinaDroolsKieServerExtension implements KieServerExtension {

	public static final String DROOLS_EXTENSION = DroolsKieServerExtension.EXTENSION_NAME;
	public static final String EXTENSION_NAME = "Drools-Mina";
	public static final String MINA_CAPABILITY = "BRM-Mina";

	private static final Logger LOG = LoggerFactory.getLogger(MinaDroolsKieServerExtension.class);

	private static final String MINA_HOST = System.getProperty("org.kie.server.drools-mina.ext.port", "localhost");
	private static final String MINA_PORT = System.getProperty("org.kie.server.drools-mina.ext.port", "9123");

	private IoAcceptor acceptor;
	private boolean initialized = false;

	@Override
	public boolean isActive() {
		return true;
	}

	@Override
	public void init(KieServerImpl kieServer, KieServerRegistry registry) {

		KieServerExtension droolsExtension = registry.getServerExtension(DROOLS_EXTENSION);
		if (droolsExtension == null) {
			LOG.warn("No Drools extension available, quiting...");
			return;
		}

		KieContainerCommandService batchCommandService = findByType(droolsExtension.getServices(),
				KieContainerCommandService.class);

		if (batchCommandService != null) {
			acceptor = new NioSocketAcceptor();
			acceptor.getFilterChain().addLast("codec",
					new ProtocolCodecFilter(new TextLineCodecFactory(Charset.forName("UTF-8"))));

			acceptor.setHandler(new TextBasedIoHandlerAdapter(batchCommandService));
			acceptor.getSessionConfig().setReadBufferSize(2048);
			acceptor.getSessionConfig().setIdleTime(IdleStatus.BOTH_IDLE, 10);
			try {
				acceptor.bind(new InetSocketAddress(MINA_HOST, Integer.parseInt(MINA_PORT)));

				LOG.info("{} -- Mina server started at {} and port {}", toString(), MINA_HOST, MINA_PORT);
			} catch (IOException e) {
				LOG.error("Unable to start Mina acceptor due to {}", e.getMessage(), e);
			}

			initialized = true;
		}
	}

	@Override
	public void destroy(KieServerImpl kieServer, KieServerRegistry registry) {
		if (acceptor != null) {
			acceptor.dispose();
			acceptor = null;
		}

		LOG.info("{} -- Mina server stopped", toString());
	}

	@Override
	public List<Object> getAppComponents(SupportedTransports type) {
		// Nothing for supported transports (REST or JMS)
		return Collections.emptyList();
	}

	@Override
	public <T> T getAppComponents(Class<T> serviceType) {

		return null;
	}

	@Override
	public String getImplementedCapability() {
		return MINA_CAPABILITY;
	}

	@Override
	public List<Object> getServices() {
		return Collections.emptyList();
	}

	@Override
	public String getExtensionName() {
		return EXTENSION_NAME;
	}

	@Override
	public Integer getStartOrder() {
		return 20;
	}

	@Override
	public String toString() {
		return EXTENSION_NAME + " KIE Server extension";
	}

	@Override
	public boolean isInitialized() {
		return initialized;
	}

	@Override
	public void createContainer(String id, KieContainerInstance kieContainerInstance, Map<String, Object> parameters) {
		// Empty, already handled by the `Drools` extension
	}

	@Override
	public void updateContainer(String id, KieContainerInstance kieContainerInstance, Map<String, Object> parameters) {
		// Empty, already handled by the `Drools` extension
	}

	@Override
	public boolean isUpdateContainerAllowed(String id, KieContainerInstance kieContainerInstance,
			Map<String, Object> parameters) {
		// Empty, already handled by the `Drools` extension
		return false;
	}

	@Override
	public void disposeContainer(String id, KieContainerInstance kieContainerInstance, Map<String, Object> parameters) {
		// Empty, already handled by the `Drools` extension
	}

	@SuppressWarnings("unchecked")
	private <T> T findByType(List<Object> services, Class<T> clazz) {
		return services.stream().filter(svc -> clazz.isAssignableFrom(svc.getClass())).map(svc -> (T) svc).findFirst()
				.get();
	}
}
```

## 5.- Make the service discoverable

To make the new endpoint discoverable, create a _src/main/resources/META-INF/services/org.kie.server.services.api.KieServerExtension_ file and add the fully qualified class name of the KieServerApplicationComponentsService implementation class within the file:

```
org.sgitario.jbpm.CustomKieServerApplicationComponentsService
```

## 6.- Build the project

```
mvn clean package
```

It will generate the file **target/new-capability-1.0-SNAPSHOT-jar-with-dependencies.jar**.

## 7.- Create our own image of Kie Server

Create a file called _Dockerfile_:

```
FROM registry.redhat.io/rhdm-7/rhdm-kieserver-rhel8:7.7.1
COPY target/new-capability-1.0-SNAPSHOT-jar-with-dependencies.jar /opt/eap/standalone/deployments/ROOT.war/WEB-INF/lib/
```

| Note that we need to sign up in [the Red Hat registry](https://access.redhat.com/RegistryAuthentication).

Then, build the image:

```
docker build . --tag quay.io/jcarvaja/rhdm-kieserver-rhel8-new-capability
```

## 8.- Prepare the KJAR example

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

## 9.- Test it

We first run our Kie Server image:

```
docker run -it -p 8080:8080 -p 9123:9123 --rm --env KIE_ADMIN_USER=admin --env KIE_ADMIN_PWD=admin --env MAVEN_MIRROR_URL=http://nexus:8081/nexus/content/groups/public/ --link nexus quay.io/jcarvaja/rhdm-kieserver-rhel8-new-capability
```

We should see this log in the server logs:

```
INFO  [org.sgitario.jbpm.MinaDroolsKieServerExtension] (ServerService Thread Pool -- 77) Drools-Mina KIE Server extension -- Mina server started at localhost and port 9123
INFO  [org.kie.server.services.impl.KieServerImpl] (ServerService Thread Pool -- 77) Drools-Mina KIE Server extension has been successfully registered as server extension
```

We're going to run the same KJAR example as in [this post](https://sgitario.github.io/jbpm-kie-server-extend-capability/) (step 7). 

```
curl -X PUT "http://admin:admin@localhost:8080/services/rest/server/containers/sayHello" -H "accept: application/json" -H "Content-Type: application/json" -d "{ \"container-id\" : \"sayHello\", \"release-id\" : { \"group-id\" : \"org.jbpm.test\", \"artifact-id\" : \"say-hello-kjar-example\", \"version\" : \"1.0-SNAPSHOT\" }}"
```

So, let's connect with the Mina server:

```
telnet 127.0.0.1 9123
```

And then type:

```
sayHello|{"lookup":"ksession","commands":[{"insert":{"object":{"org.jbpm.test.Person":{"name":"john","age":20}}}},{"fire-all-rules":""}]}
```

And, as John is older than 21, we should see the greetings in the server logs:

```
INFO  [stdout] (NioProcessor-2) Hello john
``` 

Finally, we need to type "exit" to quit the connection with the Mina server.

## Conclusion

All the source code can be found in [this repository](https://github.com/Sgitario/kie-server-examples/tree/master/kie-server-new-capability).
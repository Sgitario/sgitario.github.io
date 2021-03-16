---
layout: post
title: Docker Maven Plugin Introduction
date: 2021-03-16
tags: [ Maven ]
---

Most of the times our services need to be integrated to other services or third party components like databases. For example, our service might need a database instance to store or read books. But how can we verify our services logic? In this post, we'll see how we can use the [Docker Maven plugin](https://github.com/fabric8io/docker-maven-plugin) to start up third party services and how we can integrate it with our integration tests. 

Of course, we want to avoid manual verification/testing, instead we're looking for a test coverage that ensure our logic is fully working and also to avoid regression issues in the future. 

## Why Integration Tests?

There are several testing strategies. Among them, we can find Unit Testing and Integration Testing:
- **Unit Testing** - to ensure our logic is behaving as expected at method level. These tests are named with `*Test.java` and are executed by the [Surefire Maven plugin](https://maven.apache.org/surefire/maven-surefire-plugin/).
- **Integration Testing** - to ensure our logic is behaving as expected at service level, including the integration with third party systems. These tests are named with `*IT.java` and are executed by the [Failsafe Maven plugin](https://maven.apache.org/surefire/maven-failsafe-plugin/).

The different between each one is clear enough: our focus in unit testing is to verify every single behaviour at method level, and for doing so, we're allowed to use mocks. For integration testing is about to verify services, so we should write the tests in the most realistic way to ensure that our services work fine using third parties.

Why is so important to note this difference? Apart from that separating concerns is a good practice in testing too, because surefire and failsafe Maven plugins are designed differently to follow different purposes as we'll see in the next section.

## Docker Maven Plugin

If we don't have the failsafe plugin in our `pom.xml` file, let's add it first: 

```xml
<build>
    <plugins>
        <plugin>
            <artifactId>maven-failsafe-plugin</artifactId>
            <executions>
                <execution>
                    <goals>
                        <goal>integration-test</goal>
                        <goal>verify</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

Now, let's add the Docker Maven plugin in our `pom.xml` file:

```xml
<build>
    <plugins>
        <plugin> <!-- maven-failsafe-plugin --> </plugin>
        <plugin>
            <groupId>io.fabric8</groupId>
            <artifactId>docker-maven-plugin</artifactId>
            <configuration>
                <images>
                    <!-- ... -->
                </images>
            </configuration>
            <executions>
                <execution>
                    <id>docker-start</id>
                    <phase>pre-integration-test</phase>
                    <goals>
                        <goal>start</goal>
                    </goals>
                </execution>
                <execution>
                    <id>docker-stop</id>
                    <phase>post-integration-test</phase>
                    <goals>
                        <goal>stop</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

The failsafe Maven plugin allows to add hooks before and after the integration test execution using the `pre-integration-test` and `post-integration-test`. Here, we're configuring the plugin to start the images before running the integration tests and stopping them afterward regardless of the result of the tests (so no containers will remain up and running).

Under the images field, we configure our third party system or service using Docker. We'll see some examples later.

### Update Properties from Maven to Tests

Let's imagine that the image boots up and listens at localhost:5555. How can we configure our integration tests to use the port 5555? Again Maven makes it really simpler, let's use the resources plugin:

```xml
<properties>
    <!--Property from docker-maven-plugin -->
    <service.port>5555</service.port>
</properties>
<build>
    <testResources>
        <testResource>
            <directory>src/test/resources</directory>
            <filtering>true</filtering>
        </testResource>
    </testResources>
    <plugins>
        <plugin> <!-- maven-failsafe-plugin --> </plugin>
        <plugin> <!-- docker-maven-plugin --> </plugin>
    </plugins>
</build>
```

And having this property file in our services, `application.properties`:

```prop
database.url=localhost:${service.port}
```

Maven will automatically overwrite `${service.port}` to `5555`.

### Using Random Ports

In order to avoid conflicts, it's a good practice to use random ports. Let's see how to achieve this using the `build-helper-maven-plugin` Maven plugin:

```xml
<build>
    <testResources> <!-- see above section --> </testResources>
    <plugins>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>build-helper-maven-plugin</artifactId>
            <executions>
                <execution>
                    <id>reserve-network-port</id>
                    <goals>
                        <goal>reserve-network-port</goal>
                    </goals>
                    <phase>process-resources</phase>
                    <configuration>
                        <portNames>
                            <portName>service.port</portName>
                        </portNames>
                    </configuration>
                </execution>
            </executions>
        </plugin>
        <plugin> <!-- maven-failsafe-plugin --> </plugin>
        <plugin>
            <groupId>io.fabric8</groupId>
            <artifactId>docker-maven-plugin</artifactId>
            <configuration>
                <images>
                    <image>
                        <name>${my.service.image}</name>
                        <alias>my-service</alias>
                        <run>
                            <ports>
                                <port>service.port:5555</port>
                            </ports>
                            <wait> <!-- ... --> </wait>
                        </run>
                    </image>
                </images>
            </configuration>
            <executions> <!-- from first section --> </executions>
        </plugin>
    </plugins>
</build>
```

Now, the `build-helper-maven-plugin` Maven plugin will pick a random port and next we'll be using it to map it to 5555 when running the Docker image.

### Examples

I will write here some images that have been useful for me so far:

- **MySQL image:**

```xml
<image>
    <name>mysql:8.0.22</name>
    <alias>mysql</alias>
    <run>
        <ports>
            <port>mysql.port:3306</port>
        </ports>
        <env>
            <MYSQL_USER>test</MYSQL_USER>
            <MYSQL_PASSWORD>test</MYSQL_PASSWORD>
            <MYSQL_DATABASE>db</MYSQL_DATABASE>
            <MYSQL_ROOT_PASSWORD>test</MYSQL_ROOT_PASSWORD>
        </env>
        <log>
            <prefix>MySQL:</prefix>
            <date>default</date>
            <color>cyan</color>
        </log>
        <tmpfs>/var/lib/mysql</tmpfs>
        <wait>
            <log>.*MySQL Community Server.*</log>
            <time>20000</time>
            <!-- Kill the container, if it doesn't stop before this given time
                duration since a graceful stop was issued -->
            <kill>300000</kill>
        </wait>
    </run>
</image>
```

- **PostgreSQL image:**

```xml
<image>
    <name>postgres:13.1</name>
    <alias>postgresql</alias>
    <run>
        <env>
            <POSTGRES_USER>test</POSTGRES_USER>
            <POSTGRES_PASSWORD>test</POSTGRES_PASSWORD>
            <POSTGRES_DB>db</POSTGRES_DB>
        </env>
        <ports>
            <port>postgres.port:5432</port>
        </ports>
        <wait>
            <time>10000</time>
            <!-- Kill the container, if it doesn't stop before this given time
                duration since a graceful stop was issued -->
            <kill>300000</kill>
        </wait>
    </run>
</image>
```

- **DB2 image:**

```xml
<image>
    <name>ibmcom/db2:11.5.5.0</name>
    <alias>db2</alias>
    <run>
        <privileged>true</privileged>
        <ports>
            <port>db2.port:50000</port>
        </ports>
        <env>
            <DB2INSTANCE>test</DB2INSTANCE>
            <DB2INST1_PASSWORD>test</DB2INST1_PASSWORD>
            <DBNAME>db</DBNAME>
            <LICENSE>accept</LICENSE>
            <!-- These help the DB2 container start faster -->
            <AUTOCONFIG>false</AUTOCONFIG>
            <ARCHIVE_LOGS>false</ARCHIVE_LOGS>
            <PERSISTENT_HOME>false</PERSISTENT_HOME>
        </env>
        <log>
            <prefix>DB2:</prefix>
            <date>default</date>
            <color>cyan</color>
        </log>
        <wait>
            <log>.*Setup has completed.*</log>
            <!-- Unfortunately booting DB2 is slow, needs to set a generous (15m) timeout;
                it generally starts in 2-3 minutes, but it's been occasionally slightly above 10m -->
            <time>900000</time>
            <!-- Kill the container, if it doesn't stop before this given time
                duration since a graceful stop was issued -->
            <kill>300000</kill>
        </wait>
    </run>
</image>
```

## Conclusion

As we can see, Docker Maven plugin can be really useful and works like a charm as an alternative to [Testcontainers](https://www.testcontainers.org/).
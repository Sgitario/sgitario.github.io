---
layout: post
title: Migrating multi-module project from Gradle to Maven
date: 2025-03-25
tags: [ Java ]
---

I just went through migrating a pretty large multi-module project from Gradle to Maven, where I could learn a lot about both Maven and Gradle. Let me share in this post some lessons learned.

## Project Structure: Gradle vs Maven

If you're coming from Gradle, you're probably used to working with files like `dependencies.gradle`, `build.gradle`, `settings.gradle`, and `buildSrc`. Let's break down how these translate (or how you achieve similar functionality) in Maven.

### Gradle Structure

* **`settings.gradle`:** This file defines the subprojects in your Gradle build.
* **`dependencies.gradle`:** Gradle often uses this file to centralize dependency versions and management.
* **`buildSrc`:** This folder allows you to write custom plugins in Groovy.
* **`build.gradle`:** This is where you define your build configuration, dependencies, and plugins for each module.

### Maven Structure

Relies heavily on a multi-module setup, with a parent `pom.xml` and submodules in designated directories that inherit its configuration from the parent:

```
project/
├── pom.xml (Parent POM)
├── module-one/
│   └── pom.xml
├── module-two/
│   └── pom.xml
└── ...
```

### Migration

* From **`settings.gradle`:**

```go
buildscript {
    apply from: "dependencies.gradle"
}

dependencyResolutionManagement {
    repositories {
        mavenCentral()
    }
}

rootProject.name = "project"
include ":module-one"
include ":module-two"
```

To Maven `pom.xml` Parent POM:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.sgitario</groupId>
    <artifactId>parent</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>

    <repositories>
        <repository>
            <id>central</id>
            <name>Maven Central</name>
            <url>https://repo.maven.apache.org/maven2</url>
        </repository>
    </repositories>

    <modules>
        <module>module-one</module>
        <module>module-two</module>
    </modules>
    
  </properties>
</project>
```

* From **`dependencies.gradle`:**

```go
ext {
    springVersion="3.4.4"
    libraries = [:]
    plugins = []
}

ext.plugins = [
        "org.springframework.boot:spring-boot-gradle-plugin:${springVersion}",
]

libraries["spring-boot-dependencies"] = "org.springframework.boot:spring-boot-dependencies:${springVersion}"
```

To Maven `pom.xml` Parent POM:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.sgitario</groupId>
    <artifactId>parent</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>

    <properties>
        <!-- from ext -->
        <spring.boot.version>3.4.4</spring.boot.version>
    </properties>

    <modules>
        ...
    </modules>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring.boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <pluginManagement>
            <!-- from ext.plugins -->
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring.boot.version}</version>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
    
  </properties>
```

* From Gradle **`buildSrc`:**

In our project, we used this folder to build custom plugins that we could apply in modules to shame common dependencies and configuration:

```
buildSrc/
├── build.gradle
├── src/main/groovy/
│   └── project.common-spring-boot-conventions.gradle
```

Where `build.gradle`:

```go
buildscript {
    apply from: "../dependencies.gradle"
}

plugins {
    id "groovy-gradle-plugin"
}

repositories {
    mavenCentral()
    gradlePluginPortal()
}

dependencies {
    implementation gradleApi()

    // add all plugin artifacts to gradle classpath as defined in dependencies.gradle
    // this makes them available via plugin ID only across *all* projects
    for (pluginDependency in project.ext.plugins) {
        implementation pluginDependency
    }
}
```

And `project.common-spring-boot-conventions.gradle`:

```go
plugins {
    id "org.springframework.boot"
}

dependencies {
    implementation enforcedPlatform(libraries["spring-boot-dependencies"])

    implementation "org.springframework.boot:spring-boot-devtools"
    // ...
}
```

How to migrate the `buildSrc` custom plugins from Gradle to Maven? The answer is that Maven does not have a direct equivalent feature, but there are two approaches that serve on the same purpose in our use case above:

* Bill Of Materials (BOM)

A BOM is essentially a POM (Project Object Model) that's used to manage dependency versions. Its main job is to provide a centralized list of dependency versions, ensuring consistency across a project or set of projects.
It doesn't contain source code or build logic. Its sole purpose is version management.

* Parent

A parent module is a POM that's used to share build configurations, dependencies, and plugins among multiple modules within a project.
It can contain both dependency version management (like a BOM) and other build configurations.

* Which one to use?

Use a BOM when you need to manage dependency versions consistently across multiple projects, even if those projects are independent.
Use a parent module when you have a multi-module project and need to share build configurations among the modules of that project.

In our case, since we were using common properties/configuration, dependencies, and plugins, we decided to migrate it using a different parent POM:

In Maven `pom.xml` Parent POM:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.sgitario</groupId>
    <artifactId>parent</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>

    <modules>
        <module>spring-parent</module>
        <module>module-one</module>
        <module>module-two</module>
    </modules>
    
  </properties>
</project>
```

And in `spring-parent/pom.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <parent>
    <groupId>com.sgitario</groupId>
    <artifactId>parent</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <relativePath>../pom.xml</relativePath>
  </parent>

  <modelVersion>4.0.0</modelVersion>
  <artifactId>spring-parent</artifactId>
  <packaging>pom</packaging>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>${spring.boot.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-devtools</artifactId>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <version>${spring.boot.version}</version>
      </plugin>
    </plugins>
  </build>
</project>
```

And now, in our modules, we can choose between to inherit from the Parent POM or the Spring Parent POM.

## Dependencies and Scopes

Let's break down the differences in how you define dependencies from Gradle to Maven, and then dive into the scopes, particularly how Gradle's `implementation`, `runtimeOnly`, `api`, and `test` translate to Maven.

* `implementation` in Gradle, `compile` (default) in Maven

These dependencies are available to the module itself but it behaves differently: in Maven, these dependencies are also transitive (available to modules that depend on this one), but not in Gradle where you need to use `api` if you want to make the dependency transitive.

From Gradle:
```go
implementation "com.google.guava:guava:30.1-jre"
```

To Maven:
```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>30.1-jre</version>
</dependency>
```

* `runtimeOnly` in Gralde, `runtime` in Maven

For dependencies needed only during runtime, not during compilation.

From Gradle:
```go
runtimeOnly "org.postgresql:postgresql:42.2.18"
```

To Maven:
```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.2.18</version>
    <scope>runtime</scope>
</dependency>
```

* `testImplementation` in Gralde, `test` in Maven

For dependencies needed only during testing.

From Gradle:
```go
testImplementation "junit:junit:4.13.2"
```

To Maven:
```xml
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13.2</version>
    <scope>test</scope>
</dependency>
```

* Annotation processors

Annotation processors are powerful tools for generating code at compile time. If you're migrating from Gradle, you're likely familiar with the `annotationProcessor` configuration. In Maven, you'll be working with the `maven-compiler-plugin`. Here's how to make the switch:

From Gradle:
```go
annotationProcessor libraries["lombok"]
```

To Maven using the `maven-compiler-plugin`:

```xml
<plugin>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <parameters>true</parameters>
        <annotationProcessorPaths>
        <path>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

## Command Line Equivalents: Gradle vs. Maven

| Gradle Command                          | Maven Command                                                                           | Description                                        |
|------------------------------------------|-----------------------------------------------------------------------------------------|----------------------------------------------------|
| **`./gradlew clean`**                    | **`./mvnw clean`**                                                                      | Deletes the build directory.                      |
| **`./gradlew build`**                    | **`./mvnw package`**                                                                    | Builds the project and creates the artifact.      |
| **`./gradlew test`**                     | **`./mvnw test`**                                                                       | Runs the unit tests.                              |
| **`./gradlew install`**                  | **`./mvnw install`**                                                                    | Installs the artifact into the local repository.  |
| **`./gradlew assemble`**                 | **`./mvnw compile`**                                                                    | Compiles the source code.                         |
| **`./gradlew dependencies`**             | **`./mvnw dependency:tree`**                                                            | Displays the dependency tree.                     |
| **`./gradlew bootRun`**                  | **`./mvnw spring-boot:run`**                                                            | Run a Spring Boot application.                    |
| **`./gradlew :module-rest:bootRun`**     | **`./mvnw -pl module-rest -am spring-boot:run`**                                        | Run the module-rest Spring Boot application.      |
| **`./gradlew quarkusDev`**               | **`./mvnw quarkus:dev`**                                                                | Run a Quarkus application in Dev mode.            |
| **`./gradlew :module-rest:quarkusDev`**  | **`./mvnw -pl module-rest -am quarkus:dev`**                                            | Run the module-rest Quarkus application in Dev mode. |
| **`./gradlew tasks`**                    | **`mvn help:describe -Dplugin=org.apache.maven.plugins:maven-help-plugin -Ddetail`**    | Lists available tasks/plugins.                    |

## Plugins

Let's now go through some common well known plugins and how to migrate them.

### OpenAPI Generator

From Gradle:

```go
plugins {
    id "org.openapitools:openapi-generator-gradle-plugin:7.12.0"
}

tasks.register("generateApiV1", GenerateTask) {
    generatorName = "jaxrs-spec"
    inputSpec = "${projectDir}/my-api-spec.yaml"
    configFile = "${projectDir}/my-api-config.json"
    outputDir = "$buildDir/generated/my"
    configOptions = [
            interfaceOnly: "true",
            generatePom: "false",
            dateLibrary: "java8",
            useJakartaEe: "true",
            useTags: "true"
    ]
    importMappings = [
            "MyModel": "com.sgitario.model.MyModel",
    ]
    typeMappings = [
            "string+MyModel": "MyModel",
    ]
}

tasks.openApiGenerate.dependsOn("generateApiV1")
compileJava.dependsOn tasks.openApiGenerate
sourceSets.main.java.srcDirs += ["${buildDir}/generated/my/src/gen/java"]
```

In Maven:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>


    <build>
        <plugins>
            <plugin>
                <groupId>org.openapitools</groupId>
                <artifactId>openapi-generator-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>generateApiV1</id>
                        <goals>
                            <goal>generate</goal>
                        </goals>
                        <configuration>
                            <generatorName>jaxrs-spec</generatorName>
                            <inputSpec>${maven.multiModuleProjectDirectory}/my-api-spec.yaml</inputSpec>
                            <configurationFile>${maven.multiModuleProjectDirectory}/my-api-config.json</configurationFile>
                            <output>${project.build.directory}/generated/my</output>
                            <configOptions>
                                <interfaceOnly>true</interfaceOnly>
                                <generatePom>false</generatePom>
                                <dateLibrary>java8</dateLibrary>
                                <useJakartaEe>true</useJakartaEe>
                                <useTags>true</useTags>
                            </configOptions>
                            <importMappings>
                                <importMapping>
                                MyModel=com.sgitario.model.MyModel
                                </importMapping>
                            </importMappings>
                            <typeMappings>
                                <typeMapping>
                                string+MyModel=MyModel
                                </typeMapping>
                            </typeMappings>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

    <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>build-helper-maven-plugin</artifactId>
        <executions>
          <execution>
            <id>add-source</id>
            <phase>generate-sources</phase>
            <goals>
              <goal>add-source</goal>
            </goals>
            <configuration>
              <sources>
                <source>${project.build.directory}/generated/tally/src/gen/java</source>
              </sources>
            </configuration>
          </execution>
        </executions>
      </plugin>
</project>
```

Note that I used `${maven.multiModuleProjectDirectory}` because our API spec is defined at the root project repository. If your API spec is defined in the same project, you can use `${project.basedir}`.

### JSON Schema generator

From Gradle:

```go
plugins {
    id "jsonschema2pojo"
}

jsonSchema2Pojo {
    source = files("${projectDir}/../module-one/schemas/models.yaml")
    targetPackage = "org.sgitario.module.two.model"
    includeAdditionalProperties = false
    includeJsr303Annotations = true
    initializeCollections = false
    dateTimeType = "java.time.OffsetDateTime"
    sourceType = "yamlschema"
    generateBuilders = true
    includeGetters = true
    includeSetters = true
    useJakartaValidation = true
}
```

To Maven:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>


    <build>
        <plugins>
            <plugin>
                <groupId>org.jsonschema2pojo</groupId>
                <artifactId>jsonschema2pojo-maven-plugin</artifactId>
                <configuration>
                    <sourceDirectory>${maven.multiModuleProjectDirectory}/module-one/schemas/models.yaml</sourceDirectory>
                    <targetPackage>org.sgitario.module.two.model</targetPackage>
                    <outputDirectory>${project.build.directory}/generated/src/gen/java</outputDirectory>
                    <includeAdditionalProperties>false</includeAdditionalProperties>
                    <includeJsr303Annotations>true</includeJsr303Annotations>
                    <initializeCollections>false</initializeCollections>
                    <dateTimeType>java.time.OffsetDateTime</dateTimeType>
                    <sourceType>yamlschema</sourceType>
                    <generateBuilders>true</generateBuilders>
                    <includeGetters>true</includeGetters>
                    <includeSetters>true</includeSetters>
                    <useJakartaValidation>true</useJakartaValidation>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>build-helper-maven-plugin</artifactId>
        <executions>
          <execution>
            <id>add-source</id>
            <phase>generate-sources</phase>
            <goals>
              <goal>add-source</goal>
            </goals>
            <configuration>
              <sources>
                <source>${project.build.directory}/generated/src/gen/java</source>
              </sources>
            </configuration>
          </execution>
        </executions>
      </plugin>
</project>
```

### Spotless

From Gradle:

```go
plugins {
    id "com.diffplug.spotless"
}

spotless {
    java {
        targetExclude "**/build/**" // exclude generated code
        enforceCheck false // allows build task to be successful, even if there is a code style violation
        googleJavaFormat()
        licenseHeaderFile "${rootDir}/config/codestyle/HEADER.txt" //lets you specify code that you don't want to violate rules or be reformatted
        toggleOffOn()
    }
}
```

To Maven:

```xml
<plugin>
    <groupId>com.diffplug.spotless</groupId>
    <artifactId>spotless-maven-plugin</artifactId>
    <version>${spotless.version}</version>
    <executions>
        <execution>
        <id>validate</id>
        <phase>validate</phase>
        <goals>
            <goal>check</goal>
        </goals>
        </execution>
    </executions>
    <configuration>
        <java>
        <excludes>
            <exclude>**/target/**</exclude>
        </excludes>
        <googleJavaFormat/>
        <licenseHeader>
            <file>${maven.multiModuleProjectDirectory}/config/codestyle/HEADER.txt</file>
        </licenseHeader>
        <toggleOffOn />
        </java>
    </configuration>
</plugin>
```

Command lines:
* From `./gradlew spotlessApply` to `./mvnw spotless:apply`
* From `./gradlew spotlessCheck` to `./mvnw spotless:check`

### Checkstyle

From Gradle:

```go
plugins {
    id "checkstyle"
}

checkstyle {
    toolVersion = "10.21.4"
}

checkstyleMain {
    source = "src/main/java"
}

checkstyleTest {
    source = "src/test/java"
}
```

To Maven:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>${checkstyle.version}</version>
    <executions>
        <execution>
        <id>validate</id>
        <phase>validate</phase>
        <goals>
            <goal>check</goal>
        </goals>
        </execution>
    </executions>
    <configuration>
        <configLocation>${maven.multiModuleProjectDirectory}/config/checkstyle/checkstyle.xml</configLocation>
        <suppressionsLocation>${maven.multiModuleProjectDirectory}/config/checkstyle/checkstyle-suppressions.xml</suppressionsLocation>
        <consoleOutput>true</consoleOutput>
        <failsOnError>true</failsOnError>
        <sourceDirectories>
            <sourceDirectory>src/main/java</sourceDirectory>
            <sourceDirectory>src/test/java</sourceDirectory>
        </sourceDirectories>
    </configuration>
    <dependencies>
        <dependency>
        <groupId>com.puppycrawl.tools</groupId>
        <artifactId>checkstyle</artifactId>
        <version>10.21.4</version>
        </dependency>
    </dependencies>
</plugin>
```

To run checkstyle and generate the report in Maven: `./mvnw validate checkstyle:checkstyle`

### Jacoco

To calculate the code coverage and generate the report, there are two components:

* At the module side (where you want to calculate the code coverage after running the tests)

In Gradle:

```go
plugins {
    id "plugin"
}
```

To Maven:

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <executions>
        <execution>
        <id>prepare-agent</id>
        <goals>
            <goal>prepare-agent</goal>
        </goals>
        </execution>
    </executions>
</plugin>
```

The plugin will generate the `jacoco.exec` file with all the data.

* The report aggregator

This component will aggregate all the `jacoco.exec` files and generate the final reports in different formats, by default: HTML, XML, CSV.

In Gradle:

```go
plugins {
    id "jacoco-report-aggregation"
}

dependencies {
    subprojects.findAll { !Set.of(
            project(":exclude-this-module-from-report"),
    ).contains(it)}.each {
        jacocoAggregation it
    }
}

tasks.check.dependsOn tasks.testCodeCoverageReport
```

First, we need to create a new module in the Maven `pom.xml` Parent POM called `coverage-report` and we want to build/generate the report only when the property `coverage` is provided:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.sgitario</groupId>
    <artifactId>parent</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>

    <profiles>
    <profile>
      <id>coverage</id>
      <activation>
        <property>
          <name>coverage</name>
        </property>
      </activation>

      <modules>
        <module>coverage-report</module>
      </modules>
    </profile>
  </profiles>
    
  </properties>
</project>
```

The `pom.xml` needs to declare all the dependencies from where Jacoco will collect the `jacoco.exec`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>com.sgitario</groupId>
    <artifactId>parent</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <relativePath>../pom.xml</relativePath>
  </parent>

  <artifactId>coverage-report</artifactId>

  <build>
    <plugins>
      <plugin>
        <groupId>org.jacoco</groupId>
        <artifactId>jacoco-maven-plugin</artifactId>
        <executions>
          <execution>
            <id>report</id>
            <phase>verify</phase>
            <goals>
              <goal>report-aggregate</goal>
            </goals>
            <configuration>
              <dataFileIncludes>
                <dataFileInclude>**/jacoco.exec</dataFileInclude>
              </dataFileIncludes>
              <outputDirectory>${project.reporting.outputDirectory}/jacoco-aggregate</outputDirectory>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>

  <dependencies>
        <dependency>
            <groupId>com.sgitario</groupId>
            <artifactId>module-one</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>com.sgitario</groupId>
            <artifactId>module-two</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>com.sgitario</groupId>
            <artifactId>module-rest</artifactId>
            <version>${project.version}</version>
        </dependency>
        ...
  </dependencies>
</project>
```

And now, we can run the tests and calculate the code coverage in one go using `./mvnw clean verify -Dcoverage`.
The report will be at `coverage-report/target/site/jacoco-aggregate`.

**For Quarkus Only**: There is a Quarkus Jacoco extension that already configures all the special handling Quarkus does internally. So, the `coverage` profile from above should look like for Quarkus services as:

```xml
<profile>
    <id>coverage</id>
    <activation>
        <property>
            <name>coverage</name>
        </property>
    </activation>

    <dependencies>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-jacoco</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <executions>
                <execution>
                    <id>prepare-agent</id>
                        <goals>
                            <goal>prepare-agent</goal>
                        </goals>
                    <configuration>
                        <exclClassLoaders>*QuarkusClassLoader</exclClassLoaders>
                        <append>true</append>
                    </configuration>
                </execution>
                </executions>
            </plugin>
            <plugin>
            <artifactId>maven-surefire-plugin</artifactId>
            <configuration>
                <systemPropertyVariables>
                    <quarkus.jacoco.data-file>${project.build.directory}/jacoco.exec</quarkus.jacoco.data-file>
                    <quarkus.jacoco.reuse-data-file>true</quarkus.jacoco.reuse-data-file>
                    <quarkus.jacoco.report>false</quarkus.jacoco.report>
                </systemPropertyVariables>
            </configuration>
            </plugin>
        </plugins>
    </build>
</profile>
```

Explanation: by default, the Quarkus Jacoco extension will generate a `quarkus-jacoco.exec` file, where the coverage-report project expects a `jacoco.exec` file. We can configure this extension to generate `jacoco.exec` instead, also not to create the report (since we're going to generate it ourselves), and to reuse the same data file in case we have tests that are not using the `@QuarkusTest` annotation. Moreover, we need also to exclude the class loader `QuarkusClassLoader` to not pollute the jacoco data file with the internally generated classes by Quarkus.

## Final Hints

### Migrating unconventional project structures

Where Gradle is super flexible in how to define a project structure, this might lead into following bad patterns when designing the project structure. In Maven, this is much more restrictive, so if you're coming from Gradle, you might have to deal about how to deal with this situation. 

To be more specific, in the project we were migrating, the parent was at the same time parent, and a REST service. In Maven, you can't have the same POM.xml to serve multiple purposes. Solutions:

* Refactor the project structure to follow more conventional structures
* Create a "virtual" module:

In Maven `pom.xml` Parent POM:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.sgitario</groupId>
    <artifactId>parent</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>

    <modules>
        <module>spring-parent</module>
        <module>module-one</module>
        <module>module-two</module>
        <module>module-rest</module> <!-- new -->
    </modules>
    
  </properties>
</project>
```

And in `module-rest/pom.xml`, we specify where the sources really are:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>com.sgitario</groupId>
    <artifactId>spring-parent</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <relativePath>../spring/parent/pom.xml</relativePath>
  </parent>

  <artifactId>module-rest</artifactId>

  <build>
    <sourceDirectory>../src/main/java</sourceDirectory>
    <testSourceDirectory>../src/test/java</testSourceDirectory>
    <resources>
      <resource>
        <directory>../src/main/resources</directory>
      </resource>
    </resources>
    <testResources>
      <testResource>
        <directory>../src/test/resources</directory>
      </testResource>
    </testResources>
</project>
```

### Use relativePath when declaring the parents

When defining a parent POM in your module's `pom.xml`, you'll often see the `relativePath` element. You might wonder, "Why is this necessary?" Well, it's more than just a formality; it's critical for ensuring Maven picks up the correct parent POM, especially when you're actively working on a multi-module project.

Maven tries to find the parent POM in the following order:
1.  The local repository (your `.m2` directory).
2.  The remote repositories (like Maven Central).
3.  The file system, based on the `relativePath`.

Without `relativePath`, Maven will first look in your local repository. If a parent POM with the same `groupId`, `artifactId`, and `version` is found there (perhaps from a previous build), Maven will use that cached version.
This can lead to serious issues during development. If you've made changes to the parent POM, Maven won't see them because it's using the cached version. You'll be scratching your head, wondering why your changes aren't taking effect!
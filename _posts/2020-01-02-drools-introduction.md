---
layout: post
title: Drools Introduction
date: 2020-01-02
tags: [ java, drools ]
---

This post introduces a basic installation and example of [Drools](https://docs.jboss.org/drools/release/6.5.0.Final/drools-docs/html_single/). 

| Drools is a Business Rules Management System (BRMS) solution. It provides a core Business Rules Engine (BRE), a web authoring and rules management application (Drools Workbench), full runtime support for Decision Model and Notation (DMN) models. 

From [https://www.drools.org/](https://www.drools.org/)

## Concepts

Basically, Drools allows to write business rules following this format:

```drl
rule R when
        $r : Result()
        $p1 : Person(name == "Jose")
        $p2 : Person(name != "Jose", age > $p1.age)
    then
        $r.setValue($p2.getName() + "is older than " + $p1.getName());
end
```

After having a set of business rules registered in the Drools Workbench - or server - and after an event is fired, then the [RETE](https://en.wikipedia.org/wiki/Rete_algorithm) engine within Drools will choose and execute one of these rules.

## Learn by example

1.- Configure the maven dependencies in your *pom.xml*:

- *drools-core*: this is the core engine, runtime component that also contains the RETE engine. This is the only runtime dependency if you are pre-compiling rules (and deploying via Package or RuleBase objects).

```xml
<dependency>
    <groupId>org.drools</groupId>
    <artifactId>drools-core</artifactId>
    <version>7.31.0.Final</version>
</dependency>
```

- *drools-compiler*: this contains the compiler/builder components to take rule source, and build executable rule bases. This is often a runtime dependency of your application, but it need not be if you are pre-compiling your rules. 

```xml
<dependency>
    <groupId>org.drools</groupId>
    <artifactId>drools-compiler</artifactId>
    <version>7.31.0.Final</version>
</dependency>
```

- *kie-ci* and *kie-api*: these dependencies allow to run Drools locally and to work with Maven to resolve the rule files and other artifacts outside of the classpath.

```xml
<dependency>
    <groupId>org.kie</groupId>
    <artifactId>kie-api</artifactId>
    <version>7.31.0.Final</version>
</dependency>

<dependency>
    <groupId>org.kie</groupId>
    <artifactId>kie-ci</artifactId>
    <version>7.31.0.Final</version>
</dependency>
```

2.- Create the resource folders for your rules:

```plain
src/main/resources
    | META-INF
    | rules
```

3.- Create our rule and module:

- *resources/rules/older.drl*:

```drl
package rules

import com.sgitario.drools.model.Person

rule "Older" when
        $p1 : Person(name == "Jose")
        $p2 : Person(name != "Jose", age > $p1.age)
    then
        System.out.println($p2.getName() + " is older than " + $p1.getName());
end
```

- *resources/META-INF/kmodule.xml*:

```xml
<kmodule xmlns="http://www.drools.org/xsd/kmodule">
    <kbase name="rules" packages="rules">
        <ksession name="ksession-rules"/>
    </kbase>
</kmodule>
```

4.- And run our application:

```java
public class App {
    public static void main(String[] args) {
        try {
            // load up the knowledge base
            KieServices ks = KieServices.Factory.get();
            KieContainer kContainer = ks.getKieClasspathContainer();
            KieSession kSession = kContainer.newKieSession("ksession-rules");

            // go !
            kSession.insert(new Person("Jose", 20));
            kSession.insert(new Person("Irene", 19));
            kSession.insert(new Person("Victor", 21));
            kSession.fireAllRules();
        } catch (Throwable t) {
            t.printStackTrace();
        }
    }
}
```

Configure the manifest using Maven, *pom.xml*:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>exec-maven-plugin</artifactId>
            <version>1.4.0</version>
            <configuration>
                <mainClass>com.sgitario.drools.App</mainClass>
            </configuration>
        </plugin>
    </plugins>
</build>
```

Output:

```
mvn clean package exec:java
> Victor is older than Jose
```

## Conclusions

This post was based on [this site](http://www.mastertheboss.com/jboss-jbpm/drools/drools-and-maven-example-project), so greetings to them. Also, we worked with the *Kie* dependencies which are useful for testint purposes, but in order to take fully advantage of Drools, the rules must be integrated to [Business Manager workbench](https://docs.jboss.org/drools/release/6.2.0.CR3/drools-docs/html/wb.Workbench.html).

The source code can be found [here](https://github.com/Sgitario/drools_tutorial).
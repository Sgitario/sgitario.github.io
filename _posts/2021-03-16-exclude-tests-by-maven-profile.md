---
layout: post
title: Exclude Tests by Maven Profile
date: 2021-03-16
tags: [ Maven ]
---

There is a super useful tip in Maven to fully overwrite plugin configuration by Maven profiles using `combine.self`. In the following example, we'll see how we can exclude tests depending on the Maven profile we're running:

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
                <configuration>
                    <includes>
                        <include>**/*IT.java</include>
                    </includes>
                    <excludes>
                        <exclude>**/Special*IT.java</exclude>
                    </excludes>
                </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

Let's add now a profile for special to run these tests:

```xml
<profiles>
    <profile>
        <id>special</id>
        <build>
            <plugins>
                <plugin>
                    <artifactId>maven-failsafe-plugin</artifactId>
                    <executions>
                        <execution>
                            <configuration>
                                <excludes combine.self="override"/>
                            </configuration>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

I can't be simpler! 
Note that I've used failsafe, but this can be done in the surefire plugin as well.
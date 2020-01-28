 (Quarkus: Supersonic. Subatomic. Java.
Author: Jan Martiska (@janmartiska)

Quarkus was designed to integrate standards and well known frameworks (via extensions) in one platform with really fast startup times.

## Combine both Reactive and imperative development:

```java
@Inject
MyService say;

@GET
@Produces(MediaType.TEXT_PLAIN)
public String hello() {
    return say.hello();
}

// or

@Inject @Stream("kafka")
Publisher<String> say;

@GET
@Produces(MediaType.EVENTS...)
public String hello() {
    return say.hello(); // this is not complete
}
```

## Build-time initialization

Quarkus moves what frameworks normally do at compile time, to build time, so the app can start much faster.

- Parse config files
- Validate the application (dependencies)
- Classpath & classes scanning
- Build framework metamodel objects (@Repository in Spring)
- Prepare reflection and build proxies
- Start and open IO, threads, etc

Quarkus transforms all the above configuration and records it directly as bytecode:

- STATIC_INIT: results serialized into the native binary
- RUNTIME_INIT: bytecode is executed at boot time

## The dark side of AOT compilation (GraalVM constraints)

Not supported:
- Dynamic class loading
- Security manager
- JMX
- Finalizers, serialization

Partially supported:
- Reflective operations (it can be controlled by GraalVM argument - important if you're writting a quarkus extension)
- Dynamic proxies
- All static initializers are executed eagerly by default
- Debugging (-g requires the Enterprise Edition)

## Custom Extension by Matej Novotny and Martin Kouba

Source Code: https://github.com/mkouba/devconf2020
Documentation: https://quarkus.io/guides/writing-extensions and https://quarkus.io/quarkus-workshops/super-heroes/#extension

An extension might be needed for:
- Analyze application classes and annotations and collect metadata
- Generate and transform bytecode (there are lot of tools to do this really easily)
- Set up the runtime environment based on the configuration and metadata
- Optimize the application based on the "closed world" premise
- Workaround the GraalVM native image limitations

Three Phases of Bootstrap:
1.- Augmentation: at build time, build steps and build items
2.- Static init: at build time/runtime, with bytecode recording
3.- Runtime init: at runtime, with bytecode recording

Build Steps
- Consume and produce build items
- Automatically wired together and executed to produce the final build artifacts
- Parameters are injected
- Only called if producing something that another consumer

* Implementations are concrete, final subclasses of io.quarkus.builder.item.BuildItem class
* Should be immutable
* SimpleBuildItem - may only be produced once
* MultiBuildItem - may be produced multiple times, and consumed as a List

### Steps
1.- Add FeatureBuildItem for outputs your extension and verify it's working

```java
package org.acme.watcher.deployment;

import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.FeatureBuildItem;

public class WatcherBuildSteps {
    
    @BuildStep
    FeatureBuildItem feature() {
        return new FeatureBuildItem("watcher");
    }

}
```

Then, in our example quarkus app that uses our extension, we should see:

```
> mvn compile quarkus:dev
2020-01-26 12:19:34,141 INFO  [io.quarkus] (main) Profile dev activated. Live Coding activated.
2020-01-26 12:19:34,141 INFO  [io.quarkus] (main) Installed features: [cdi, resteasy, watcher]
```

2.- Configuration: let's make the extension configurable

```java
package org.acme.watcher;

import io.quarkus.runtime.annotations.ConfigItem;
import io.quarkus.runtime.annotations.ConfigPhase;
import io.quarkus.runtime.annotations.ConfigRoot;

@ConfigRoot(phase = ConfigPhase.BUILD_AND_RUN_TIME_FIXED)
public class WatcherConfig {

    /**
     * Regex that defines what JAX-RS resource methods should be watched automatically based on the resource class to be
     * included.
     * <p>
     * By default, asterisk(*) is used, meaning all resources will be modified.
     */
    @ConfigItem(defaultValue = ".*")
    public String regularExpression;

    /**
     * The time limit used by watcher. If a method invocation exceeds the limit a CDI even of type {@link LimitExceeded} is
     * fired.
     * <p>
     * By default, the limit is 500ms.
     */
    @ConfigItem(defaultValue = "500")
    public long limit;

}
```

This configuration exposes a couple of properties that we can use as:

*application.properties*:

```
quarkus.watcher.limit=600
quarkus.watcher.regular-expression=org.(first|third).*
```

3.- Index Analysys

Analysis is done via Jandex index (?): immutable index of all classes in your deployment.
We're going to analyze the deployment for JAX-RS resources, then create custom build items from what we find.

```java
package org.acme.watcher.deployment;

import java.lang.reflect.Modifier;

import javax.ws.rs.GET;

import org.acme.watcher.WatcherConfig;
import org.jboss.jandex.AnnotationInstance;
import org.jboss.jandex.AnnotationTarget.Kind;
import org.jboss.jandex.DotName;
import org.jboss.jandex.IndexView;
import org.jboss.jandex.MethodInfo;

import io.quarkus.arc.deployment.BeanArchiveIndexBuildItem;
import io.quarkus.deployment.annotations.BuildProducer;
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.FeatureBuildItem;

public class WatcherBuildSteps {
    
    // ... feature()
    
    // THIS IS NEW!
    @BuildStep
    void collectResourceMethods(BeanArchiveIndexBuildItem beanArchive,
            BuildProducer<WatchedResourceMethodBuildItem> resourceMethods,
            WatcherConfig config) {

        IndexView index = beanArchive.getIndex();
        DotName getDotName = DotName.createSimple(GET.class.getName());

        for (AnnotationInstance annotation : index.getAnnotations(getDotName)) {
            if (annotation.target().kind() == Kind.METHOD) {
                MethodInfo method = annotation.target().asMethod();
                // filter out methods based on config expression
                if (Modifier.isPublic(method.flags())
                        && method.declaringClass().name().toString().matches(config.regularExpression)) {
                    resourceMethods.produce(new WatchedResourceMethodBuildItem(method)); // important
                }
            }
        }
    }

}
```

And our new build item:

```java
package org.acme.watcher.deployment;

import org.acme.watcher.WatcherInterceptor;
import org.jboss.jandex.MethodInfo;

import io.quarkus.builder.item.MultiBuildItem;

/**
 * {@link WatcherInterceptor} is automatically bound to a JAX-RS resource method represented by this build item.
 */
public final class WatchedResourceMethodBuildItem extends MultiBuildItem {

    private final MethodInfo method;

    public WatchedResourceMethodBuildItem(MethodInfo method) {
        this.method = method;
    }

    public MethodInfo getMethod() {
        return method;
    }

}

```

4.- Transform Annotations

At the time, the build step is not executed because no component is consuming/expecting this new build item. 

```java
package org.acme.watcher.deployment;

import java.lang.reflect.Modifier;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

import javax.ws.rs.GET;

import org.acme.watcher.Watch;
import org.acme.watcher.WatcherConfig;
import org.jboss.jandex.AnnotationInstance;
import org.jboss.jandex.AnnotationTarget.Kind;
import org.jboss.jandex.DotName;
import org.jboss.jandex.IndexView;
import org.jboss.jandex.MethodInfo;

import io.quarkus.arc.deployment.AnnotationsTransformerBuildItem;
import io.quarkus.arc.deployment.BeanArchiveIndexBuildItem;
import io.quarkus.arc.processor.AnnotationsTransformer;
import io.quarkus.deployment.annotations.BuildProducer;
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.FeatureBuildItem;

public class WatcherBuildSteps {
    
    // ... feature()
    
    // .... collectResourceMethods()
    
    @BuildStep
    AnnotationsTransformerBuildItem transformAnnotations(List<WatchedResourceMethodBuildItem> resourceMethods) {

        DotName watchDotName = DotName.createSimple(Watch.class.getName());
        Set<MethodInfo> methods = resourceMethods.stream().map(WatchedResourceMethodBuildItem::getMethod)
                .collect(Collectors.toSet());

        return new AnnotationsTransformerBuildItem(new AnnotationsTransformer() {

            public boolean appliesTo(Kind kind) {
                return Kind.METHOD.equals(kind);
            }

            @Override
            public void transform(TransformationContext context) {
                if (methods.contains(context.getTarget())) {
                    context.transform().add(watchDotName).done();
                }
            }
        });
    }

}
```

5.- Register Interceptor

Still it doesn't work because we need to registeran interceptor.


```java
public class WatcherBuildSteps {
    
    // ... feature()
    
    // ... collectResourceMethods()
    
    // ... transformAnnotations()

    @BuildStep
    AdditionalBeanBuildItem registerBeans() {
        return AdditionalBeanBuildItem.builder().addBeanClasses(WatcherInterceptor.class, Watcher.class).build();
    }

}
```

6.- How to test this?

```java
package org.acme;

import static org.junit.jupiter.api.Assertions.assertEquals;

import javax.inject.Inject;

import org.hamcrest.CoreMatchers;
import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.asset.StringAsset;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import io.quarkus.test.QuarkusUnitTest;
import io.restassured.RestAssured;

public class ResourceInterceptionTest {

    @RegisterExtension
    static QuarkusUnitTest test = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class).addClasses(TestResource.class, EventsCollector.class)
                    .addAsResource(new StringAsset("quarkus.watcher.limit=100"), "application.properties"));

    @Inject
    EventsCollector collector;

    @Test
    public void testResourceGetsIntercepted() {
        RestAssured.given()
                .when().get("/test")
                .then()
                .statusCode(200)
                .body(CoreMatchers.containsString("ping"));

        RestAssured.given()
                .when().get("/test/limit")
                .then()
                .statusCode(200)
                .body(CoreMatchers.containsString("ping"));

        assertEquals(1, collector.getEvents().size());
        assertEquals("org.acme.TestResource#limitExceeded()", collector.getEvents().get(0).methodInfo);
    }

}

```

Where *TestResource* is:

```java
package org.acme;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Path("/test")
public class TestResource {

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String ping() {
        return "ping";
    }

    @GET
    @Path("/limit")
    @Produces(MediaType.TEXT_PLAIN)
    public String limitExceeded() {
        try {
            Thread.sleep(200l);
        } catch (InterruptedException e) {
            throw new IllegalStateException(e);
        }
        return "ping";
    }

}
```

7.- Byte Recorder

```java
package org.acme.watcher;

import java.util.Set;
import java.util.stream.Collectors;

import org.jboss.logging.Logger;

import io.quarkus.runtime.annotations.Recorder;

@Recorder
public class WatcherRecorder {

    private static final Logger LOGGER = Logger.getLogger(WatcherRecorder.class);

    public void summarizeBootstrap(WatcherConfig config, Set<String> affectedMethods) {
        LOGGER.infof("\n" +
                " _       __ ___   ______ ______ __  __ ______ ____ \n" +
                "| |     / //   | /_  __// ____// / / // ____// __ \\\n" +
                "| | /| / // /| |  / /  / /    / /_/ // __/  / /_/ /\n" +
                "| |/ |/ // ___ | / /  / /___ / __  // /___ / _, _/ \n" +
                "|__/|__//_/  |_|/_/   \\____//_/ /_//_____//_/ |_|  \n" +
                "                                                   \n" +
                "\n" +
                "" +
                "Regular expression: [%s]\n" +
                "Interceptor threshold limit: %s ms\n" +
                "Monitored JAX-RS methods: \n\t- %s\n",
                config.regularExpression, config.limit, affectedMethods.stream().collect(Collectors.joining("\n\t- ")));
    }
}

```

```java
public class WatcherBuildSteps {
    
    // ... feature()
    
    // ... collectResourceMethods()
    
    // ... transformAnnotations()

    // ... registerBeans()

    @BuildStep
    @Record(ExecutionTime.RUNTIME_INIT)
    void recordAffectedMethods(List<WatchedResourceMethodBuildItem> resourceMethods, WatcherRecorder recorder,
            WatcherConfig config) {
        // use the recorder to summarize config and affected methods
        recorder.summarizeBootstrap(config, resourceMethods.stream().map(
                buildItem -> buildItem.getMethod().declaringClass().toString() + "#" + buildItem.getMethod().name())
                .collect(Collectors.toSet()));
    }

}
```
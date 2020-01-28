# Kogito Workshop

We're going to implement a *Travel Agency Process" (attach diagram from the mobile with main process and then the subprocesses - another picture).

Instructions: https://github.com/cristianonicolai/devconfcz-2020

## Requirements
- Maven
- Java 11
- VS Code
- Install the Kogito Extension in VSCode

| Download the latest Visual Studio plugin from the project page: https://github.com/kiegroup/kogito-tooling/releases
| 1. Select the latest version
| 2. From asset section download the file vscode_extension_kogito_kie_editors_0.2.7.vsix (the version might change)
| 3. Open Visual Studio Code
| 4. Select the Extensions pane on the left
| 5. Click the... icon on the top right
| 6. Select Install from VSIX...

There is also a chrome extension where we can edit the diagrams directly from the browser.

## Instructions

1- Create Maven project

```bash
mvn io.quarkus:quarkus-maven-plugin:1.1.1.Final:create -DprojectGroupId=org.acme.travel -DprojectArtifactId=kogito-travel-agency -Dextensions="kogito,openapi"
```

This command will create a quarkus application with the kogito and openapi extensions.

2.- Build and run the application

```bash
cd kogito-travel-agency
mvn compile quarkus-dev
```

Output:

```bash
2020-01-25 15:46:07,051 INFO  [io.quarkus] (main) kogito-travel-agency 1.0-SNAPSHOT (running on Quarkus 1.1.1.Final) started in 2.651s. Listening on: http://0.0.0.0:8080
2020-01-25 15:46:07,054 INFO  [io.quarkus] (main) Profile dev activated. Live Coding activated.
2020-01-25 15:46:07,055 INFO  [io.quarkus] (main) Installed features: [cdi, kogito, resteasy, resteasy-jackson, smallrye-openapi, swagger-ui]
```

3.- Clone the sources

TODO: use my github account instead
```
git clone https://github.com/cristianonicolai/devconfcz-2020
cp -r devconfcz-2020/src/* kogito-travel-agency/src/

cd kogito-travel-agency
mvn clean compile
```

Run the test rules: Add the following dependency in the pom.xml file in the project root just after the tag <dependencies>:

```xml
<dependency>
      <groupId>org.kie.kogito</groupId>
      <artifactId>drools-decisiontables</artifactId>
</dependency>
```

```bash
mvn clean verify
```

4.- Check the decision tables: visa-rules.xls (I need to provide more info about this - maybe this is part of the Drools post)

5.- Create services:

```java
package org.acme.travel.service;

import java.util.Date;

import javax.enterprise.context.ApplicationScoped;

import org.acme.travel.Flight;
import org.acme.travel.Trip;

/**
 * FlightBookingService
 */
@ApplicationScoped
public class FlightBookingService {

    public Flight bookFlight(Trip trip) {
        return new Flight("MX555", "1A", "16A", new Date(), new Date());
    }
}
```

And:

```java
package org.acme.travel.service;

import javax.enterprise.context.ApplicationScoped;

import org.acme.travel.Address;
import org.acme.travel.Hotel;
import org.acme.travel.Trip;

/**
 * HotelBookingService
 */
@ApplicationScoped
public class HotelBookingService {

    public Hotel bookHotel(Trip trip) {
        return new Hotel("Perfect hotel", new Address("street", trip.getCity(), "12345", trip.getCountry()), "09876543", "XX-012345");
    }
}
```

These two services will be used in the subprocesses (second image).

6.- Create subprocesses:

Create hotel.bpmn under src/main/resources/package../:

TODO: More
---
layout: post
title: Kogito - Travel Agency Example
date: 2020-04-01
tags: [ kogito ]
---

The Travel Agency application is becaming the most relevant and complete example in the Kogito workshops and tutorials so far. This is because it makes use of every single component in the Kogito ecosystem since UI, to Data Index or Jobs Service. 

![Architecture]({{ site.url }}{{ site.baseurl }}/images/kogito_travel_1.png)

Basically, the application will check whether a person needs a VISA to travel. If so, the request will be on hold until the agency submit the VISA for this person. Then, it will book the hotel and the flight. Finally, it will require a user confirmation (somebody from the travel agency) to proceed.

In this tutorial, we'll see how to create this example from zero using VS Code (so we can use the latest BMPN editor from [Kogito Tooling](https://github.com/kiegroup/kogito-tooling).

## Requirements
- [Maven](https://maven.apache.org/)
- Java 11
- [VS Code](https://code.visualstudio.com/)

### Kogito Extension in VS Code

Download the latest assets from https://github.com/kiegroup/kogito-tooling/releases

1. Select the latest version
2. From asset section download the file [vscode_extension_kogito_kie_editors_0.X.vsix](https://github.com/kiegroup/kogito-tooling/releases/download/0.2.10/vscode_extension_kogito_kie_editors_0.2.10.vsix)
3. Open Visual Studio Code
4. Select the Extensions pane on the left
5. Click the... icon on the top right
6. Select Install from VSIX...

| There is also a chrome extension where we can edit the diagrams directly from the browser.

## Let's start!

1. Create the Maven project

```sh
mvn io.quarkus:quarkus-maven-plugin:1.3.1.Final:create \
    -DprojectGroupId=org.acme.travel \
    -DprojectArtifactId=kogito-travel-agency \
    -Dextensions="kogito,openapi"
cd kogito-travel-agency
```

This command will create a quarkus application with the next extensions:
- [Kogito](https://quarkus.io/guides/kogito): to enable Kogito capabilities
- [Openapi](https://quarkus.io/guides/openapi-swaggerui): to expose its API description through an OpenAPI specification using [Swagger UI](https://swagger.io/).

Let's first build and run the application:

```sh
mvn compile quarkus-dev
```

Expected output:

```sh
2020-01-25 15:46:07,051 INFO  [io.quarkus] (main) kogito-travel-agency 1.0-SNAPSHOT (running on Quarkus 1.3.1.Final) started in 2.651s. Listening on: http://0.0.0.0:8080
2020-01-25 15:46:07,054 INFO  [io.quarkus] (main) Profile dev activated. Live Coding activated.
2020-01-25 15:46:07,055 INFO  [io.quarkus] (main) Installed features: [cdi, kogito, resteasy, resteasy-jackson, smallrye-openapi, swagger-ui]
```

2. Think on business first

All about Kogito and Business automation is about to think on businesses first based on rules. So, let's create a decision table or spreadsheet, so we can decide when a traveller requires VISA or not.

![Decision Table]({{ site.url }}{{ site.baseurl }}/images/kogito_travel_decision_table.png)

| Pay attention to the RuleSet, Import and the conditions. This is what Drools engine uses to trigger the rules.

Basically, if we are Polish and we go to either US or Australia, we require VISA. Otherwise, if we go to UK, we don't. 

Let's add this spreadsheet as **visa-rules.xls** into *src/main/resources/org/acme/travels*. Example of this file can be found [here](https://github.com/cristianonicolai/devconfcz-2020/blob/solved/src/main/resources/org/acme/travel/visa-rules.xls).

Also, we need to add the following dependency in the *pom.xml*:

```xml
<dependency>
      <groupId>org.kie.kogito</groupId>
      <artifactId>drools-decisiontables</artifactId>
</dependency>
```

And this is how we make use of it in the diagram:

![Visa Component]({{ site.url }}{{ site.baseurl }}/images/kogito_travel_visas.png)

3. Prepare the model Java classes

All the below classes must be created in the package *org.acme.travels*.

*Traveller.java*:
```java
public class Traveller {

	private String firstName;
	private String lastName;
	private String email;
	private String nationality;
    private Address address;
    
    // constructors, getters and setters
}
```

*Address.java*:
```java
public class Address {

	private String street;
	private String city;
	private String zipCode;
    private String country;
    
    // constructors, getters and setters
}
```

*Trip.java*:
```java
public class Trip {

	private String city;
	private String country;
	private Date begin;
	private Date end;
    private boolean visaRequired;

    // constructors, getters and setters
}
```

*VisaApplication.java*:
```java
public class VisaApplication {

	private String firstName;
	private String lastName;
	private String city;
	private String country;
	private int duration;
	private String passportNumber;
	private String nationality;
    private boolean approved;

    // constructors, getters and setters
}
```

*Flight.java*:
```java
public class Flight {

	private String flightNumber;
	private String seat;
	private String gate;
	private Date departure;
    private Date arrival;

    // constructors, getters and setters
}
```

*Hotel.java*:
```java
public class Hotel {
	
	private String name;
	private Address address;
	private String phone;
	private String bookingNumber;
    private String room;
    
    // constructors, getters and setters
}
```

4. The "Book Hotel" and "Book Flight" services

Let's divide the problem in smaller units. Looking at the diagram, we'll work out in this area:

![Book Hotel and Flight]({{ site.url }}{{ site.baseurl }}/images/kogito_travel_book_area.png)

- The "Book Hotel" component

This component will call to the "bookingHotel" diagram as we can see in the properties:

![Book Hotel]({{ site.url }}{{ site.baseurl }}/images/kogito_travel_book_hotel_calling.png)

So, we need to create another diagram called *bookingHotel.bpmn2* with id "bookingHotel":

![Book Hotel]({{ site.url }}{{ site.baseurl }}/images/kogito_travel_book_hotel.png)

And this component will invoke the service *org.acme.travels.service.HotelBookingService* and method "bookHotel". Let's create it:

```java
@ApplicationScoped
public class HotelBookingService {

	public Hotel bookHotel(Trip trip) {
		return new Hotel("Perfect hotel", new Address("street", trip.getCity(), "12345", trip.getCountry()), "09876543", "XX-012345");
	}
}
```

| Pay attention to the assignments. This is how we wire all the components each other.

- The "Book Flight" component

For this component, the same as we did for "Book Hotel", but it will use this service instead:

```java
@ApplicationScoped
public class FlightBookingService {

	public Flight bookFlight(Trip trip) {
		return new Flight("MX555", trip.getBegin(), trip.getEnd());
	}
}
```

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
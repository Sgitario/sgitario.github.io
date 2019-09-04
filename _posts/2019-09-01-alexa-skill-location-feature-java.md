---
layout: post
title: Location support for Alexa Skills in Java
date: 2019-09-04
tags: [ Alexa, Java ]
---

I wrote [my first tutorial](https://sgitario.github.io/alexa-skills-notes/) for creating a Skill in Alexa almost a year ago. Recently, I wanted to create a skill that provide the conditions of the most closest beach according to your current location. How can we get the user location? I found many sources but all of them were written in node.js. Let's use Java 8 here.

> First all, we cannot test the location services in either the Alexa Simulator or devices that do not have location features.

## 1.- Enable Permission from Skill developer console

Go to [the developer console](https://developer.amazon.com/alexa), click Edit for your skill page to open your skill and select Build > Permissions to enable the Location Services button.

![Alexa Location permission]({{ site.url }}{{ site.baseurl }}/images/alexa-skill-location.png)

Without doing so, your skill cannot ask the customer for permissions to use location information.

## 2.- Modify your AWS Lambda service

In summary, we need to check whether your device is compatible with geolocation, then check whether we have the permission to get the user location and finally, proceed with the received location or ask for the "playa".

```java
private static final String GEO_PERMISSION = "alexa::devices:all:geolocation:read";

@Override
public Optional<Response> handle(HandlerInput input) {
    ResponseBuilder response = input.getResponseBuilder();

    if (deviceSupportsGeolocation(input)) {
        return handleAtCurrentLocation(input, response);
    } else {
        return handleAskForLocation(response);
    }
}

private Optional<Response> handleAskForLocation(ResponseBuilder response) {
    return say("¿Qué playa?", response);
}

private Optional<Response> handleAtCurrentLocation(HandlerInput input, ResponseBuilder response) {
    if (hasGeolocationPermission(input)) {
        return say(getCurrentLocation(input).map(this::searchBeach)
                .orElse("No pudimos obtener tu localización, ¿qué playa?"), response);
    } else {
        askForPermissionIfDenied(response);
        return say("No tenemos permiso para obtener tu localización. Concede el permiso o dinos qué playa.",
                response);
    }
}

private Optional<Response> say(String speechText, ResponseBuilder response) {
    response.withSpeech(speechText).withSimpleCard("Playa", speechText).withReprompt(speechText);
    return response.build();
}

private boolean deviceSupportsGeolocation(HandlerInput input) {
    return input.getRequestEnvelope().getContext().getSystem().getDevice().getSupportedInterfaces()
            .getGeolocation() != null;
}

private boolean hasGeolocationPermission(HandlerInput input) {
    Permissions permissions = input.getRequestEnvelope().getSession().getUser().getPermissions();
    return permissions != null && permissions.getScopes() != null
            && permissions.getScopes().get(GEO_PERMISSION).getStatus() == PermissionStatus.GRANTED;
}

private void askForPermissionIfDenied(ResponseBuilder response) {
    response.withAskForPermissionsConsentCard(Arrays.asList(GEO_PERMISSION));
}

private Optional<Coordinate> getCurrentLocation(HandlerInput input) {
    GeolocationState geo = input.getRequestEnvelope().getContext().getGeolocation();
    return Optional.ofNullable(geo.getCoordinate());
}

private String searchBeach(Coordinate location) {
    // ... we use the OPEN AEMET API here.
}
```

## Summary

This tutorial has followed the official documentation [here](https://developer.amazon.com/es/docs/custom-skills/location-services-for-alexa-skills.html#configure-your-skill-to-use-location-services).

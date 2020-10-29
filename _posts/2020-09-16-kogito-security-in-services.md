---
layout: post
title: Enable Security in Kogito using Keycloak
date: 2020-09-16
tags: [ security, kogito, keycloak ]
---

Security is an important topic when we want to have our business logic and services in the cloud. In this post, we'll see how to enable the security in every single component within [Kogito](https://sgitario.github.io/kogito-introduction/).

Note that we are going to configure the components individually, however Kogito is a cloud-native solution and for cloud solutions, a possibly better solution would be to delegate the security routing and configuration into Istio. See more about this approach in [my previous post](https://sgitario.github.io/adding-security-using-istio/).

## Requirements

- [Previous Knowledge about Kogito](https://sgitario.github.io/kogito-introduction/)
- [Kogito CLI](https://sgitario.github.io/kogito-developer-guide/), step 6.
- [Keycloak](https://www.keycloak.org/) instance, we'll be using the url "https://our-keycloak-instance", the realm "kogito" and the client "kogito-client".

We're going how to configure every Kogito component using the Kogito CLI and also directly using the custom resource.

## Data Index Service

- Via Kogito CLI

```
kogito install data-index \
  --env quarkus.oidc.tenant-enabled=true \
  --env quarkus.oidc.auth-server-url=https://our-keycloak-instance \
  --env quarkus.oidc.client-id=kogito-client \
  --env quarkus.http.auth.permission.unsecure.paths=/health/* \
  --env quarkus.http.auth.permission.unsecure.policy=permit \
  --env quarkus.http.auth.permission.secure.paths=/* \
  --env quarkus.http.auth.permission.secure.policy=authenticated \
  /
```

- Using Custom Resource:

```yaml
kind: KogitoDataIndex
apiVersion: app.kiegroup.org/v1alpha1
metadata:
  name: data-index
spec:
  envs:
  - name: quarkus.oidc.client-id
    value: kogito-client
  - name: quarkus.http.auth.permission.unsecure.paths
    value: "/health/*"
  - name: quarkus.http.auth.permission.unsecure.policy
    value: permit
  - name: quarkus.http.auth.permission.secure.paths
    value: "/*"
  - name: quarkus.http.auth.permission.secure.policy
    value: authenticated
  - name: quarkus.oidc.tenant-enabled
    value: 'true'
  - name: quarkus.oidc.auth-server-url
    value: https://our-keycloak-instance
```

## Management Console

- Via Kogito CLI

```
kogito install mgmt-console \
  --env quarkus.oidc.tenant-enabled=true \
  --env quarkus.oidc.auth-server-url=https://our-keycloak-instance \
  --env quarkus.oidc.client-id=kogito-client \
  --env quarkus.http.auth.permission.unsecure.paths=/health/* \
  --env quarkus.http.auth.permission.unsecure.policy=permit \
  --env quarkus.http.auth.permission.secure.paths=/* \
  --env quarkus.http.auth.permission.secure.policy=authenticated \
  /
```

- Using Custom Resource:

```yaml
kind: KogitoMgmtConsole
apiVersion: app.kiegroup.org/v1alpha1
metadata:
  name: management-console
spec:
  envs:
  - name: quarkus.oidc.client-id
    value: kogito-client
  - name: quarkus.http.auth.permission.unsecure.paths
    value: "/health/*"
  - name: quarkus.http.auth.permission.unsecure.policy
    value: permit
  - name: quarkus.http.auth.permission.secure.paths
    value: "/*"
  - name: quarkus.http.auth.permission.secure.policy
    value: authenticated
  - name: quarkus.oidc.tenant-enabled
    value: 'true'
  - name: quarkus.oidc.auth-server-url
    value: https://our-keycloak-instance
```

## Trusty Service

- Via Kogito CLI

```
kogito install trusty \
  --env quarkus.oidc.tenant-enabled=true \
  --env quarkus.oidc.auth-server-url=https://our-keycloak-instance \
  --env quarkus.oidc.client-id=kogito-client \
  --env quarkus.http.auth.permission.unsecure.paths=/health/* \
  --env quarkus.http.auth.permission.unsecure.policy=permit \
  --env quarkus.http.auth.permission.secure.paths=/* \
  --env quarkus.http.auth.permission.secure.policy=authenticated \
  /
```

- Using Custom Resource:

```yaml
kind: KogitoTrusty
apiVersion: app.kiegroup.org/v1alpha1
metadata:
  name: trusty
spec:
  envs:
  - name: quarkus.oidc.client-id
    value: kogito-client
  - name: quarkus.http.auth.permission.unsecure.paths
    value: "/health/*"
  - name: quarkus.http.auth.permission.unsecure.policy
    value: permit
  - name: quarkus.http.auth.permission.secure.paths
    value: "/*"
  - name: quarkus.http.auth.permission.secure.policy
    value: authenticated
  - name: quarkus.oidc.tenant-enabled
    value: 'true'
  - name: quarkus.oidc.auth-server-url
    value: https://our-keycloak-instance
```

## Jobs Service

```
kogito install jobs-service \
  --env quarkus.oidc.tenant-enabled=true \
  --env quarkus.oidc.auth-server-url=https://our-keycloak-instance \
  --env quarkus.oidc.client-id=kogito-client \
  --env quarkus.http.auth.permission.unsecure.paths=/health/* \
  --env quarkus.http.auth.permission.unsecure.policy=permit \
  --env quarkus.http.auth.permission.secure.paths=/* \
  --env quarkus.http.auth.permission.secure.policy=authenticated \
  /
```

- Using Custom Resource:

```yaml
kind: KogitoJobsService
apiVersion: app.kiegroup.org/v1alpha1
metadata:
  name: jobs-service
spec:
  envs:
  - name: quarkus.oidc.client-id
    value: kogito-client
  - name: quarkus.http.auth.permission.unsecure.paths
    value: "/health/*"
  - name: quarkus.http.auth.permission.unsecure.policy
    value: permit
  - name: quarkus.http.auth.permission.secure.paths
    value: "/*"
  - name: quarkus.http.auth.permission.secure.policy
    value: authenticated
  - name: quarkus.oidc.tenant-enabled
    value: 'true'
  - name: quarkus.oidc.auth-server-url
    value: https://our-keycloak-instance
```

## Kogito Runtime

Here, we have the option to implement our services using Quarkus or Spring Boot. We can see an example of each in the [Kogito examples](https://github.com/kiegroup/kogito-examples) repository.

### Using Quarkus

Example can be found [here](https://github.com/kiegroup/kogito-examples/tree/stable/process-usertasks-with-security-oidc-quarkus).

### Using Spring Boot

Example can be found [here](https://github.com/kiegroup/kogito-examples/tree/stable/process-usertasks-with-security-oidc-springboot).

## Appendix: Using a Keycloak instance with invalid SSL certificate

Obviously, either using a Quarkus or Spring, we need that our Keycloak instance runs using a valid SSL certificate otherwise it fails. However, we can turn off this SSL validation. Note that this is a very bad practice only suitable for demo purposes. The only additional property we need to append is:

- For Quarkus (it applies all the Kogito Services and Kogito Runtime using Quarkus):

```
quarkus.oidc.tls.verification=none
```

- For Spring Boot (only for Kogito Runtime using Spring Boot):

```
keycloak.disable-trust-manager=true
```
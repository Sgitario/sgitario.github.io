---
layout: post
title: Kogito - Developer Guide
date: 2020-03-30
tags: [ kogito, operators ]
---

Steps:

0. Requirements
- [Openjdk 11](https://tecadmin.net/install-java-on-fedora/) devel package:
- [Openshift CLI](https://docs.openshift.com/container-platform/4.2/cli_reference/openshift_cli/getting-started-cli.html)
- [Go 1.13](https://golang.org/)
- [Node.JS](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm)
- [Quay](https://quay.io) account
- An Openshift instance and project

1. Build [Kogito Runtimes](https://github.com/kiegroup/kogito-runtimes)

```sh
git clone https://github.com/kiegroup/kogito-runtimes
cd kogito-runtimes
mvn clean install -DskipTests
```

2. Build [Kogito Apps](https://github.com/kiegroup/kogito-apps)

```sh
git clone https://github.com/kiegroup/kogito-apps
cd kogito-apps
npm install -D yarn
yarn install
mvn clean install -DskipTests
```

3. Prepare Your Openshift Project

4. Install Third Party Operators

As we are going to install the operator manually from sources, we need to install the third party operators by ourselves (when installing the operator from OperatorHub, this is not needed).

- Install operator group:

```sh
export OC_NAMESPACE=mynamespace # your openshift project 

cat > operatorGroup.yaml << EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: $OC_NAMESPACE
  namespace: $OC_NAMESPACE
spec:
  targetNamespaces:
  - $OC_NAMESPACE
EOF
oc apply -f operatorGroup.yaml
rm operatorGroup.yaml
```

- [Infinispan Operator](https://infinispan.org/infinispan-operator):

```sh
export OPERATOR_PACKAGE=infinispan
export OPERATOR_CHANNEL=stable
cat > infinispan.yaml << EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: $OPERATOR_PACKAGE
  namespace: $OC_NAMESPACE
spec:
  name: $OPERATOR_PACKAGE
  source: community-operators
  sourceNamespace: openshift-marketplace
  channel: $OPERATOR_CHANNEL
EOF
oc apply -f infinispan.yaml
rm infinispan.yaml
```

- [Strimzi Operator (Kafka)](https://strimzi.io/):

```sh
export OPERATOR_PACKAGE=strimzi-kafka-operator
export OPERATOR_CHANNEL=stable
cat > strimzi.yaml << EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: $OPERATOR_PACKAGE
  namespace: $OC_NAMESPACE
spec:
  name: $OPERATOR_PACKAGE
  source: community-operators
  sourceNamespace: openshift-marketplace
  channel: $OPERATOR_CHANNEL
EOF
oc apply -f strimzi.yaml
rm strimzi.yaml
```

5. Build and Deploy [Kogito Cloud Operator](https://github.com/Sgitario/kogito-cloud-operator)

First, we need to install some prerequisities:

- [Operator SDK](https://github.com/operator-framework/operator-sdk) V0.16.0 or later
- [Golint dependency](golang.org/x/lint/golint): go get -u golang.org/x/lint/golint

Then, build the operator:

```sh
git clone https://github.com/Sgitario/kogito-cloud-operator
cd kogito-cloud-operator
make build
```

Deploy the operator into [Quay](https://quay.io):

```sh
make deploy version=0.9.0 quay_user=OUR_USER quay_pwd=PWD quay_namespace=jcarvaja
```

Expected output:

```sh
Operator is now in quay.io/jcarvaja/kogito-cloud-operator:0.9.0
```

| Check whether the operator is now in your Quay account and **make the repository public** so Openshift can reach it out.

Finally, run the Operator into Openshift:

```sh
oc login youropenshiftinstance.com # login
oc project mynamespace # select openshift project
make deploy-operator-on-ocp image=quay.io/jcarvaja/kogito-cloud-operator:0.9.0
```

6. Install Services

First, we need to install Kogito CLI:

```sh
cd kogito-cloud-operator
make install-cli
```

And, then the services:

```sh
kogito use-project mynamespace # your openshift project 
kogito install data-index
kogito install jobs-service
kogito install mgmt-console
```

And, let's finish deploying an example:

```
kogito deploy-service example-drools https://github.com/kiegroup/kogito-examples --context-dir drools-quarkus-example
```

# Appendix. Add Latest Images from Source:

In case we want to override the images that the operator is using to deploy the components, we can do it this way:

```
git clone https://github.com/kiegroup/kogito-images
cd kogito-images
oc apply -f kogito-imagestream.yaml
```
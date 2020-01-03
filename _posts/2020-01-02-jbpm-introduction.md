---
layout: post
title: jBPM Introduction
date: 2020-01-03
tags: [ java, jbpm ]
---

Following the Decision Manager series on my blog site, in this post we introduce the basics for [jBPM](https://www.jbpm.org/) (Java Business Process Management). jBPM allows to write our business logic using flow charts on the latest [BPMN 2.0](https://www.omg.org/spec/BPMN/2.0/) standard specification. We'll be doing the same example as our previous [post](https://sgitario.github.io/drools-introduction/).

The underlying environment in jBPM allows devs to deploy/execute business processes and business analysts to check what is going on with the solution. 

## Installation

jBPM has an Eclipse plugin and web based tool to edit the flow charts of your business. However, we'll be using the web based approach along this post. 

So, first thing we need to do is to deploy a jBPM server via docker-compose (image from [here](https://hub.docker.com/r/jboss/jbpm-server-full)):

*docker-compose.yml*:

```yml
version: '2'

volumes:
  mysql_data:
    driver: local

services:
  mysql:
    image: mysql:5.7
    volumes:
    - mysql_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: jbpm
      MYSQL_USER: jbpm
      MYSQL_PASSWORD: jbpm
  jbpm:
    image: jboss/business-central-workbench-showcase
    environment:
      JBPM_DB_DRIVER: mysql
      JBPM_DB_HOST: mysql
    ports:
      - 8080:8080
      - 8001:8001
    depends_on:
      - mysql
  kie-server:
    image: jboss/kie-server-showcase
    environment:
      KIE_SERVER_LOCATION: http://kie-server:8080/kie-server/services/rest/server
      KIE_SERVER_CONTROLLER: http://jbpm:8080/business-central/rest/controller
      KIE_MAVEN_REPO: http://jbpm:8080/business-central/maven2
    ports:
      - 8180:8080
    depends_on:
      - jbpm
```

And start it up:

```bash
docker-compose up
```

Now, we have our jBPM server with a persistent storage in MySQL. We can browse our server in [http://localhost:8080/business-central](http://localhost:8080/business-central) and use the predefined user admin/admin.

![Business Central]({{ site.url }}{{ site.baseurl }}/images/jbpm-bussiness-central-1.png)

We have deployed the next artifacts:

- MySQL: the persistent storage of our data
- [Kie Server](https://docs.jboss.org/drools/release/6.2.0.Final/drools-docs/html/ch19.html): the executing server that will instantiate and execute the rules.
- [jBPM Workbench](https://docs.jboss.org/jbpm/v6.0/userguide/wb.Workbench.html): this is the drool workbench we talked in the [Drool post](https://sgitario.github.io/drools-introduction/).

## Learn by example

As we said in the introduction, we'll be doing the same example in the previous post: a rule to decide which person is older than Jose.

Starting from the business central, let's create a new project by selecting *projects*.

![Create Project]({{ site.url }}{{ site.baseurl }}/images/jbpm-bussiness-central-2.png)

Then, name the project "Older Age":

![Create Project]({{ site.url }}{{ site.baseurl }}/images/jbpm-bussiness-central-3.png)

And click on *Add*.

We have created our project, so let's move on by defining our flowchart. The first thing, we'll be doing is adding a couple of assets:

![Add Asset]({{ site.url }}{{ site.baseurl }}/images/jbpm-bussiness-central-4.png)

- The Person Data Object:

![Add Data Object]({{ site.url }}{{ site.baseurl }}/images/jbpm-bussiness-central-7.png)

![Add Data Object]({{ site.url }}{{ site.baseurl }}/images/jbpm-bussiness-central-6.png)

- The drool rule:

![Add Rule]({{ site.url }}{{ site.baseurl }}/images/jbpm-bussiness-central-5.png)

And let's create our rule (more about Drool rules in [here](https://sgitario.github.io/drools-introduction/)):

![Add Rule]({{ site.url }}{{ site.baseurl }}/images/jbpm-bussiness-central-8.png)

### Test our Project

We need to add a new asset called "Test Scenario" and provide a couple of scenarios:

![Add Rule]({{ site.url }}{{ site.baseurl }}/images/jbpm-bussiness-central-9.png)

The first scenario is to check whether we detect older persons and the second scenario is the opposite. 

If we run the test, it will fail since our rule is not setting the *isOld* field in the rule so far. Let's fix it! We need modify the rule asset we added previously:

![Updated Rule]({{ site.url }}{{ site.baseurl }}/images/jbpm-bussiness-central-10.png)

Now, if we save and go back to the test scenario, if we click on the run scenarios, the test will pass now.

### Deploy our Project

Finally, we can build and deploy our Project in our execution server: 

![Deploy Project]({{ site.url }}{{ site.baseurl }}/images/jbpm-bussiness-central-11.png)

After we click on *Deploy*, we should see the next in the logs:

```
jbpm_1        | 11:46:39,657 INFO  [org.kie.server.controller.websocket.notification.WebSocketNotificationService] (default task-16) WebSocket notification about change requested on server ServerTemplateKey{id='kie-server-4c3043051b7e', name='kie-server-4c3043051b7e'} with container spec ContainerSpec{releasedId=com.myspace:OlderAge:1.0.0-SNAPSHOT, configs={RULE=org.kie.server.controller.api.model.spec.RuleConfig@c81de3a, PROCESS=org.kie.server.controller.api.model.spec.ProcessConfig@bd2a4c03}, status=STARTED} ContainerSpecKey{id='OlderAge_1.0.0-SNAPSHOT', containerName='OlderAge', serverTemplateKey=ServerTemplateKey{id='kie-server-4c3043051b7e', name='kie-server-4c3043051b7e'}} with following result [Container{serverInstanceId='kie-server-4c3043051b7e@kie-server:8080', resolvedReleasedId=com.myspace:OlderAge:1.0.0-SNAPSHOT, messages=[org.kie.server.api.model.Message@6ce5dbd], status=STARTED} ContainerKey{serverTemplateId='kie-server-4c3043051b7e', containerSpecId='OlderAge_1.0.0-SNAPSHOT', containerName='null', url='http://kie-server:8080/kie-server/services/rest/server/containers/OlderAge_1.0.0-SNAPSHOT'}]
```

Note the project url: http://kie-server:8080/kie-server/services/rest/server/containers/OlderAge_1.0.0-SNAPSHOT which we'll use to run our project via curl:

```bash
curl --location --request POST 'http://localhost:8180/kie-server/services/rest/server/containers/instances/OlderAge_1.0.0-SNAPSHOT' \
--header 'Content-Type: application/json' \
--header 'X-KIE-ContentType: JSON' \
-u 'kieserver:kieserver1!' \
--data-raw '{
      "lookup" : null,
      "commands" : [ {
        "insert" : {
	          "object": {
		          "com.myspace.olderage.Person": {
		            "name": "Jose",
		            "age": 25
		          }
		        }
	        }
        },
        {
        "insert" : {
          "object": {
	          "com.myspace.olderage.Person": {
	            "name": "Victor",
	            "age": 26
	          }
	        }
        }
 
      }, {
        "fire-all-rules" : { }
      } ]
    }'
```

Response:

```
{
  "type" : "SUCCESS",
  "msg" : "Container OlderAge_1.0.0-SNAPSHOT successfully called.",
  "result" : {
    "execution-results" : {
      "results" : [ ],
      "facts" : [ ]
    }
  }
}
```

If we are using Postman, the parameter "-u" is the basic authentication.

## BPMN

We have seen a very basic example so far. To take the full advantage of jBPM, we need to be familiar with BPMN modelling:

<iframe src="https://drive.google.com/file/d/1lc6QE6M_afIEUzQNaDWbiyFIle9ahno6/preview" width="640" height="480"></iframe>

And then, use the *Process* assets.

## Conclusions

Personally it made me hard to start using all these applications. However, in order to take full advantage of these amazing tools, there are more features that were out of scope for this post and we'll revisit soon: 

- Pluggable persistence and transactions based on JPA / JTA.
- Pluggable human task service based on WS-HumanTask for including tasks that need to be performed by human actors.
- Management console supporting process instance management, task lists and task form management, and reporting.
- Optional process repository to deploy your process (and other related knowledge).
- History logging (for querying / monitoring / analysis).
- Integration with various frameworks such as CDI/EJB, Spring(Boot), OSGi, etc.

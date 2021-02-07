---
layout: post
title: Java EE Basics
date: 2021-02-07
tags: [ java ]
---

I've enrolled in a full course learning path that requires certifications to achieve. The first one is about `Programming in Java EE`. Moreover, sometimes, It's good to recap with some basics concepts around something you're already familiar with. In the end, you always will learn something you have missed or forgotten. 

# Programming in Java EE

From simple Java SE command line application into a multi-tiered enterprise application using various Java EE specifications including:
- Enterprise Java Beans
- Java Persistence API
- Java Messaging Service
- JAX-RS for REST services
- Contexts
- Dependency Injection
- Security

## Transitioning to Multi-tiered Applications

- Enterprise applications are characterized by their ability to handle transactional workloads, multi-component integration, security, distributed architectures, and scalability.
- Compare the features of Java EE to Java SE. **Java EE** specification is a set of APIs built on top of Java SE. It provides a runtime environment for running multi-threaded, transactional, secure and scalable enterprise applications. An application server that passes a test suite called the Technology Compatibility Kit (TCK) for Java EE is known as a Java EE compliant application server. Java EE includes support for multiple profiles, or subsets of APIs. For example, the Java EE 7 specification defines two profiles: the full profile and the web profile.
- Java EE 7 has two profiles `full` and `web`. APIs part of Java EE 7: 

```
Batch API
Java API for JSON Processing (JSON-P)
Concurrency utilities
WebSocket API
Java Messaging Service (JMS)
Java Persistence API (JPA)
Java Connector Architecture (JCA)
Java API for RESTful web services (JAX-RS)
Java API for XML web services (JAX-WS)
Servlet API
Java Server Faces (JSF)
Java Server Pages (JSP)
Contexts and Dependency Injection (CDI)
Java Transaction API (JTA)
Enterprise Java Beans (EJB)
Bean Validation API
```

- The Java Community Process:

The Java Community Process (JCP) is an open, participatory process to develop, maintain, and revise Java technology specifications. The JCP fosters evolution of the Java platform in cooperation with the international Java developer community.

Any individual or organization can join the JCP and participate in the standardization process. The JCP manages the specification of a large number of APIs using Java Specification Requests (JSR).

Members are free to submit proposals for JSRs for any Java APIs. The proposal is reviewed by the JCP Executive Committee (EC), which consists of several senior Java community leaders elected by the JCP members. Once approved, a proposal is managed by a team of JCP members called as an Expert Group (EG).

The expert group is responsible for defining, finalizing, and maintaining the specification of the API under the leadership of a Specification Lead (SL). When the JSR is ready for release, it is approved by the executive committee and becomes a JCP standard. Each JSR can be incrementally evolved and refined based on the needs of Java developers and current technology trends.

All individual APIs that constitute the Java EE 7 specification, itself managed in a separate JSR, have been developed under the JCP process and have individual JSRs managed by separate expert groups specializing in a particular technology domain.

The JCP, in collaboration with the expert groups, is also responsible for publishing a Technology Compatibility Kit (TCK), a test suite that verifies whether an implementation conforms to the specification. For example, an application server is said to be "Java EE 7 compatible" only if it passes the Java EE 7 TCK fully and completely, without any errors or failures.

- Describe various multi-tiered architectures. The advantage of using tiered architectures is that as the application scales to handle more and more end-users, each of the tiers can be independently scaled to handle the increased workload by adding more servers (a process known as "scale out"). There is also the added benefit that components across tiers can be independently upgraded without impacting other components.

```
web-centric architecture
Business-to-Business (B2B)
```

## Packaging and Deploying a Java EE Application

- An application server is a software component that provides the necessary runtime environment and infrastructure to host and manage Java EE enterprise applications. Red Hat JBoss Enterprise Application Platform 7, JBoss EAP 7, or simply EAP, and is based on the Wildfly open source software. Concepts in EAP:

```
modules: versioned apps (it includes the EAP core)
containers: interface between application components and low-level infra services. Ex: Web or EJB containers.
```

- The Java Naming and Directory Interface (JNDI) is a Java API for a directory service (for looking up remote or not resources) that allows components to discover and look up objects via a logical name. Example:

```xml
<subsystem xmlns="urn:jboss:domain:datasources:4.0">
<datasources>
   <datasource jndi-name="java:jboss/datasources/ExampleDS" pool-name="ExampleDS" enabled="true" use-java-context="true">
       <connection-url>jdbc:h2:mem:test;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE</connection-url>
       <driver>h2</driver>
       <security>
           <user-name>sa</user-name>
           <password>sa</password>
       </security>
   </datasource>
</datasources>
<!-- Another example -->
<jms-queue name="helloWorldQueue" entries="java:jboss/jms/queue/helloWorldQueue"/>
```

```java
public class TestDS {

    @Resource(name="java:jboss/datasources/ExampleDS")
    private javax.sql.DataSource ds;

    @Resource(mappedName = "java:jboss/jms/queue/helloWorldQueue")
    private Queue helloWorldQueue;

    // Use the DataSource reference to create a Connection etc..
```

- Packaging and Deploying Java EE Applications

```
jar: java code, XML deployment descriptors, or code-level annotations
war: jar files, XML deployment descriptor files under the WEB-INF or WEB-INF/classes/META-INF folders
ear: jar files, war files, XML deployment descriptors in the META-INF folder
```

- Maven deploys applications to JBoss EAP using the Wildfly Maven plug-in, which provides features to deploy and undeploy applications to EAP. It supports deploying all three types of deployment units: JAR, WAR, and EAR.

## Creating Enterprise Java Beans

- Enterprise Java Beans (EJB): where to implement the application business logic. There are two types: 

```
session (sync): stateless, stateful, singleton
message driven bean (MDB) (async):
```

- Access an EJB both locally and remotely. Example:

```java
@Stateless
public class TodoBean {
// ...
}
```

```java
public class TodoClient {

  @EJB
  TodoBean todo;
  //invoke EJB methods
}
```

Remotely, an common interface must exist between the client and the server:

```java
@Stateless
@Remote(Calculator.class)
public class CalculatorBean implements Calculator {
// ...
}
```

```java
public class CalculatorClient {

  public static void main(String[] args) throws Exception {
    // /<application-name>/<module-name>/<bean-name>!<fully-qualified-interface-name>
    String JNDI_URL= "myapp/calculator-ejb/CalculatorBean!com.sgitario.training.ejb.Calculator";

    try {
      Context ic = new InitialContext();
      Calculator calc = (Calculator) ic.lookup(JNDI_URL);
      System.out.println("Response from server = " + calc.add(1,2);
      ...
    }
    catch (Exception e) {
      // handle the exception
    }

  }
}
```

You also need to provide a file called jndi.properties in the class path of the client program, with the host name, IP address, port, and security details (if secured for remote access) of the remote application server where the EJB is running.

```
java.naming.factory.initial=org.jboss.naming.remote.client.InitialContextFactory
java.naming.provider.url=http-remoting://10.2.0.15:8080
jboss.naming.client.ejb.context=true
```

- Describe the life cycle of EJBs:

For Stateful:
```
Does Not Exist -> Ready -> Passivated
```

For Stateless:
```
Does Not Exist -> Ready -> Bean Pool (Inactive)
```

For Singleton:
```
Does Not Exist -> Ready
```

All can be controlled via annotations `@PostConstruct`, `@PreDestroy`, ...

- Describe container-managed and bean-managed transactions and demarcate each in an EJB. Two ways:

Implicit or Container Managed Transaction (CMT): all automatic. Can be controlled using annotations `@TransactionAttribute(TransactionAttributeType.REQUIRED or .REQUIRES_NEW or .MANDATORY or .NOT_SUPPORTED or .SUPPORTS or .NEVER)`

Explicit or Bean Managed Transaction (BMT): managed by devs. Example:

```java
@Stateless
@TransactionManagement(TransactionManagementType.BEAN)
public class BeanManagedEJB {

    @Inject
    private UserTransaction tx;

    public Integer saveOrder(Order order) {
      try {
            tx.begin();

            Integer orderId = createOrder(order);
            raisePurchaseOrder(orderId);

            sendEMail("Your order # " + orderId + "has been placed successfully..." );

            tx.commit();

            return orderId;
        } catch (Exception e) {
            tx.rollback();
            return null;
        }
```

## Managing Persistence

- Describe the Persistence API: Java EE provides the Java Persistence API (JSR 338) specification that is implemented by various ORM providers. There are many ORM software offerings available in the market, such as EclipseLink and Hibernate. A fully implemented ORM provides optimization techniques, caching, database portability, query language in addition to object persistence. The three key concepts related to the Java Persistence API are entities, persistence units, and persistence context.

- Describing Entity Manager:

```java
@Stateless
public class ItemService {
  //ItemPU is the name of the persistence unit
    EntityManagerFactory emFactory = Persistence.createEntityManagerFactory("ItemPU");
    EntityManager em = emFactory.createEntityManager();
    ....
}
```

or

```java
public class EMProducer {
     @Produces
     @PersistenceContext(unitName= "ItemPU")
     private EntityManager em;
}
```

And then:

```java
@Stateless
public class ItemService{
   @Inject
   private EntityManager em;

   public void registerItem(Item item) throws Exception {
     ...
     em.persist(item);
     ....
   }
   public void removeItem(Long id) throws Exception {
     ...
     em.remove(findById(id));
     ....
   }
   public void updateItem(Item item) {
    em.merge(item);
   }
}
```

- Describing Persistence Unit: A Persistence unit is configured in a persistence.xml file in the application's META-INF directory. 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="2.1"
	xmlns="http://xmlns.jcp.org/xml/ns/persistence" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
	http://xmlns.jcp.org/xml/ns/persistence/persistence_2_1.xsd">
	<persistence-unit name="Items" transaction-type="JTA">
		<jta-data-source>java:jboss/datasources/MySQLDS</jta-data-source>
		<properties>
			<property name="hibernate.dialect" value="org.hibernate.dialect.MySQLDialect" />
			<property name="hibernate.hbm2ddl.auto" value="update" />
			<property name="hibernate.show_sql" value="true" />
			<property name="hibernate.format_sql" value="true" />
		</properties>
	</persistence-unit>
</persistence>
```

- Types of transactions: Resource Local Transactions (single data source) and JTA Transactions (multiple within the same container).

- Persist data to a data store using entities. Entity states: New, Managed, Removed, Detached

- EntityManager Interface and Key Methods: persist(), find(), contains(), merge(), remove(), clear(), refresh()

- Annotate beans to validate data. Manual invocation:

```java
Person p = new Person();
p.setPersonName("RH");
ValidatorFactory validatorFactory = Validation.buildDefaultValidatorFactory();
Validator validator = validatorFactory.getValidator();
Set<ConstraintViolation<Person>> constraintViolations = validator.validate(p);
for (ConstraintViolation<Person> cv : constraintViolations) {
    log.error(cv.getMessage());
}
```

- Create a query using the Java Persistence Query Language:

```java
Query query=entityManager.createQuery("SELECT e from Employee e");
List<Employee> persons = query.getResultList();

TypedQuery<Employee> query=entityManager.createQuery("SELECT e from Employee e where e.salary >?1 or e.empName=?2", Employee.class);

Query query=entityManager.createQuery("SELECT e from Employee e where e.salary >:sal");
query.setParameter("sal", salary);

Query query=entityManager.createQuery("SELECT e from Employee e where e.salary >?1");
query.setParameter(1, salary);
```

## Managing Entity Relationships

- Configure one-to-one and one-to-many entity relationships: @OneToOne, @OneToMany, @ManyToOne, @ManyToMany, @JoinColumn
- Using the Join Fetch Clause in JPQL Queries to Eagerly Fetch Related Entities:

```java
TypedQuery<Make> query = em.createQuery("SELECT ma FROM Make ma JOIN FETCH ma.models WHERE ma.id = :id" , Make.class);
```

- Describe many-to-many entity relationships.

## Creating REST Services

- Describe JAX-RS RESTful Web Services: JAX-RS is the Java API for creating lightweight RESTful web services. In Red Hat JBoss EAP 7, the implementation of JAX-RS is RESTEasy, which is fully compliant with the JSR-311 specification entitled Java API for RESTful Web Services 2.0 and provides additional features for efficient development of REST services.

```java
@ApplicationPath("/api")
public class Service extends Application {
	// ...
}
```

or in web.xml:

```xml
<web-app version="3.0" xmlns="http://java.sun.com/xml/ns/javaee"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd">
  <servlet>
      <servlet-name>javax.ws.rs.core.Application</servlet-name>
  </servlet>
  <servlet-mapping>
      <servlet-name>javax.ws.rs.core.Application</servlet-name>
      <url-pattern>/api/*</url-pattern>
  </servlet-mapping>
  ...
</web-app>
```

Services:

```java
@Path("hello")
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
public class HelloWorld {

  @GET
  public String hello() {
     return "Hello World!";
  }
}
```

Clients:

```java
Client client = ClientBuilder.newClient();
WebTarget webTarget = client.target("http://localhost:8080/todo/api/users");

// paths
WebTarget items = webTarget.path("items");

// headers
String encodedCred = new BASE64Encoder().encode(userpass.getBytes());
itemTarget.request().header("Authorization", "Basic " + encodedCred);

// methods
Response response = webTarget.request().get();
Response response = webTarget.request().post(Entity.json(item));

// response
Item foundItem = response.readEntity(Item.class);

// close client
client.close();
```

- Describe JAX-WS Web Services: JAX-WS is the Java API for XML-based web services using the Simple Object Access Protocol (SOAP). JBossWS is the JSR-224 Java API for XML-based Web Services 2.2 specification compliant implementation for JAX-WS in Red Hat JBoss EAP 7. To define a standard protocol for communication between applications, JAX-WS services use an XML definition file written using Web Services Description Language (WSDL).

## Implementing Contexts and Dependency Injection

The Contexts and Dependency Injection (CDI) specification is one of many subordinate specifications within the Java EE specification. While CDI was introduced in Java EE 6, the concepts behind CDI have been around in various frameworks including Spring, Google Guice, and others. The Java Community Process introduced Java Specification Request 299 in its final form in December 2009. JSR 346 formally defines CDI for the Java EE 7 platform. This means that every application server that is certified as Java EE 7 compliant, such as JBoss EAP, must support contexts and dependency injection natively.

- Using Dependency Injection: @Inject, using Qualifiers, using Producers

- Comparing resource injection and CDI:

CDI is similar to using resource injection to inject resources such as a @PersistenceContext and a persistence.xml file. Both approaches create resource dependencies managed by the container, and both loosely couple application components. They do, however, differ in several important ways. Because resource injection uses the JNDI name to inject a resource, resource injection is not type safe like CDI. CDI is type safe because objects are instantiated based on type. Further, CDI is able to inject regular Java classes directly, while resource injection cannot inject regular classes and instead refers to the resource by JNDI names.

- Comparing EJB and CDI

Differentiating EJB and CDI is important because there is an overlap in features between the two specifications. In Java EE 7 applications built to run on JBoss EAP, it is common for developers to use both technologies in conjunction with each other. All EJBs are CDI beans, and therefore have access to dependency injection and are eligible to be injected themselves.

The EJB specification builds upon the CDI specification and provides even more functionality, distinguishing between stateless and stateful beans. EJBs also provide other functionality such as concurrency features, bean pooling, security, as well as others that are not included in CDI. When creating a bean, it is a good practice to not use an EJB if the features of an EJB are not required. Instead, use CDI for managing contexts and dependency injection.

- Apply scopes to beans appropriately: Request, Session, Conversation, Application, Singleton
- Naming Beans: @Named

### Naming Beans

The @Named annotation provides a mechanism for EJBs to be referenced using expression language (EL) found in front-end libraries, such as Java Server Faces (JSF) pages.

## Creating Messaging Applications with JMS

Messaging technology or message-oriented-middleware (MOM) is an essential tool used by many developers in enterprises around the world. By enabling asynchronous processing between loosely-coupled systems, messaging allows developers to build applications that are more efficient, more easily scaled, and more reliable.

The Java EE specification defines the Java Message Service (JMS) API in order to create a standardized method for Java applications to connect to and utilize enterprise messaging solutions. JMS was first introduced in 2002 as JSR-914 and the current version used by Java EE 7, v2.0 is maintained under JSR-343.

```
JMS Client: Java programs or applications that send or receive messages.
Message: 	Data, typically in a standardized format, sent between applications or clients.
JMS Provider:	The messaging component or technology that implements the JMS API. When working with JBoss EAP, a JMS provider is built into the server, or an external provider can be configured.
Administered Object: A preconfigured JMS object defined in the container configuration and is used by an application. Typically, connection factories and destinations are administered objects that are defined at the server level and are made available by the container for use by the deployed applications.
Connection Factory: A Java object used by a client to create new connections to the JMS provider.
Destination: A Java object used by a client to specify the queue or topic where the client sends or receives messages.
```

- Using JMS on JBoss EAP 7 with the Embedded Apache Artemis Message Broker: JBoss EAP 7 deploys and manages Artemis as the messaging-activemq subsystem. After installing JBoss EAP, the only profile that enables the messaging subsystem is the standalone-full.xml configuration file. It is therefore necessary to either use this configuration file or manually enable the messaging-activemq subsystem in a custom standalone.xml configuration file:

```xml
<subsystem xmlns="urn:jboss:domain:messaging-activemq:1.0">
  <server name="default">
    ...Some configuration omitted...
    <jms-queue name="ExpiryQueue" entries="java:/jms/queue/ExpiryQueue"/>
    <jms-queue name="DLQ" entries="java:/jms/queue/DLQ"/>
    <jms-queue name="helloWorldQueue" entries="queue/helloWorldQueue java:jboss/jms/queue/helloWorldQueue"/>
    <jms-queue name="TodoListQueue" entries="queue/TodoListQueue java:jboss/jms/queue/TodoListQueue"/>1
    <connection-factory name="InVmConnectionFactory" entries="java:/ConnectionFactory" connectors="in-vm"/>2
     <connection-factory name="RemoteConnectionFactory" entries="java:jboss/exported/jms/RemoteConnectionFactory" connectors="http-connector"/>
     <pooled-connection-factory name="activemq-ra" transaction="xa" entries="java:/JmsXA java:jboss/DefaultJMSConnectionFactory" connectors="in-vm"/>
  </server>
</subsystem>
```

- Describe the components that make up the JMS API.

ConnectionFactory:

```java
@Resource(lookup = "java:jboss/ConnectionFactory")
private static ConnectionFactory connectionFactory;
```

or via JNDI:

```java
final Properties env = new Properties();
env.put(Context.INITIAL_CONTEXT_FACTORY, INITIAL_CONTEXT_FACTORY);
env.put(Context.PROVIDER_URL, System.getProperty(Context.PROVIDER_URL, PROVIDER_URL));
env.put(Context.SECURITY_PRINCIPAL, userName);
env.put(Context.SECURITY_CREDENTIALS, password);
namingContext = new InitialContext(env);

// Perform the JNDI lookups
String cfString = System.getProperty("connection.factory", DEFAULT_CONNECTION_FACTORY);
ConnectionFactory connectionFactory = (ConnectionFactory) namingContext.lookup(cfString);
```

Destination:

```java
@Resource(lookup = "jms/MyQueue")
private static Queue queue;

@Resource(lookup = "jms/MyTopic")
private static Topic topic;
```

Context:

```java
JMSContext context = connectionFactory.createContext();
```

Message Producer:

```java
JMSProducer producer = context.createProducer();
producer.send(dest, message);
```

Message Consumer:

```java
JMSConsumer consumer = context.createConsumer(dest);
Message m = consumer.receive();
```
Headers:

```
JMSMessageID: The unique identifier for each message sent by a provider, set automatically.
JMSDestination: The destination where the message is being sent.
JMSPriority: The priority level of the message. This can be used to reorder messages and advance more important messages to the top of the queue.
JMSDeliveryMode: The persistence settings of the message, controlling how the message is handled by the provider.
JMSDeliveryTime: The timestamp when the message was delivered by the JMS provider.
JMSTimeStamp: The timestamp when the message is given to the JMS provider.
JMSExpiration: The time when the message should be considered expired.
JMSCorrelationID: The ID of another message, used to relate two messages, typically set by the client.
JMSReplyTo: The destination where replies to messages can be sent.
JMSRedelivered: The status of whether or not the message is currently a re-delivery.
```

Body:

```
Message: An empty body. This type of message is only headers and properties and is used when a message body is not needed by the application.
BytesMessage: A stream of bytes, typically used to encode a body to match an existing message format.
StreamMessage: A stream of Java primitives, to be read sequentially.
TextMessage: A Java String object, which can include JSON, or XML data.
ObjectMessage: Any serializeable Java object.
MapMessage: A set of key-value pairs, where the keys are String objects and the values are Java primitives.
```

```java
TextMessage hello = context.createTextMessage();
hello.setText("Hello World!");
context.createProducer().send(hello);
```

- Create a JMS client that produces and consumes messages using the JMS API:

```java
@Resource(mappedName = "java:jboss/jms/queue/helloWorldQueue")
private Queue helloWorldQueue;

@Resource(mappedName = "java:comp/DefaultJMSConnectionFactory")
private static ConnectionFactory connectionFactory;

@Inject
@JMSConnectionFactory("jms/MyConnectionFactory")
private JMSContext context;

JMSProducer producer = context.createProducer();

TextMessage message = context.createTextMessage(msg);

producer.send(helloWorldQueue, message);

JMSConsumer consumer = context.createConsumer(helloWorldQueue);

try {
	TextMessage msg = (TextMessage) consumer.receiveNoWait();
	if(msg != null) {
		System.out.println("Received Message: "+ msg);
		return msg.getBody(String.class);
	}else {
		return null;
	}
}
catch (Exception e) {
	e.printStackTrace();
	return null;
} finally {
	consumer.close();
}

```

- Create, package, and deploy a message driven bean.

The JMS MessageConsumer interface differs from an MDB in a number of important ways. The developer must manually invoke the MessageConsumer, whereas an MDB is automatically created. In addition, the JMS consumer only allows synchronous checking for new messages and is typically single-threaded unless the developer manually implements concurrency. In contrast, the MDB is asynchronous and multi-threaded. For these reasons, MDBs are a more robust solution for Java EE applications that need to consume messages from a destination asynchronously.

```java
@MessageDriven(name = "QueueListener", activationConfig = {
  @ActivationConfigProperty(propertyName = "destinationLookup", propertyValue = "queue/helloWorldQueue"),
  @ActivationConfigProperty(propertyName = "destinationType", propertyValue = "javax.jms.Queue")
})
public class QueueListener implements MessageListener {

	private final static Logger LOGGER = Logger.getLogger(this.class.getName());

	public void onMessage(Message rcvMessage) {
		TextMessage msg = null;

		try {
			if (rcvMessage instanceof TextMessage) {
				msg = (TextMessage) rcvMessage;
				LOGGER.info("Received Message from helloWorldQueue ===> " + msg.getText());
			} else {
				LOGGER.warn("Incorrect Message Type!");
			}
		} catch (JMSException e) {
			throw new RuntimeException(e);
		}
	}
```

Where:

```
destinationLookup: The JNDI name of the queue or topic. This property is mandatory.
connectionFactoryLookup	The JNDI name of the connection factory to use to connect to the destination. This property is optional, and if not specified the pooled connection factory named activemq-ra is used.
destinationType: The type of the destination, either javax.jms.Queue or javax.jms.Topic. This property is mandatory.
messageSelector: A string used to select a subset of the available messages. The syntax is similar to SQL syntax and is described in detail in JMS specification, and is beyond the scope of this course. This property is optional.
acknowledgeMode: The type of acknowledgment when not using transacted JMS. Valid values are Auto-acknowledge or Dups-ok-acknowledge. This property is optional. If not specified, the default value is Auto-acknowledge.
```

## Securing Java EE Applications

- Describe the JAAS specification: Java Authentication and Authorization Service (JAAS) is a security API that is used to implement user authentication and authorization in Java applications (JSR-196). The API extends the Java Enterprise Edition access control architecture to support user-based authorization. Java EE enforces the API as a mechanism to enable and guarantee access. JAAS provides declarative role-based security in the JBoss Enterprise Application Platform. Declarative security separates security concerns from application code by using the container to manage security. The container provides an authorization system based on annotations and XML descriptors within the application code that secures resources. This approach is in contrast to programmatic security, which requires each application to contain code that manages security.

- Declarative Security:

In web.xml:
```xml
<security-constraint>
    <web-resource-collection>
         <web-resource-name>All resources</web-resource-name>
         <url-pattern>/*</url-pattern>
     </web-resource-collection>
     <auth-constraint>
            <role-name>*</role-name>
     </auth-constraint>
   </security-constraint>
   <login-config>
      <auth-method>BASIC</auth-method>
      <realm-name>basicRealm</realm-name>
   </login-config>
```
- Via annotations:

```
@SecurityDomain: Located at the beginning of the class, this annotation defines the security domain by name to use for the EJB.

@DeclareRoles: Located at the beginning of the class, this annotation defines the roles that are tested for permissions in the class. If this annotation is not used, the roles are checked based on the presence of the @RolesAllowed annotations.

@RolesAllowed: Located either at the beginning of the class or before a method header, this annotation defines a list of one or more roles allowed to access a method. If placed before the class header, methods in the class without an annotation default to this annotation.

@PermitAll: Located either at the beginning of the class or before a method header, this annotation specifies that all roles are allowed to access a method.

@DenyAll: Located either at the beginning of the class or before a method header, this annotation specifies that no roles are allowed to access a method.

@RunAs: Located either at the beginning of the class or before a method header, this annotation specifies the role used when running a method. This annotation is useful when an EJB is calling another EJB and needs to take on a new role for security restrictions in another EJB.
```

Example:
```java
@Stateless
@RolesAllowed({"admin, qa"})
@SecurityDomain("other") 
public class HelloWorldEJB implements HW {
    @PermitAll
    public String HelloWorld(String msg) {
        return "Hello " + msg;
    }
    @RolesAllowed("admin")
    public String GoodbyeAdmin(String msg) {
        return "See you later, " + msg;
    }
	
    public String GoodbyeSecure(String msg) {
        return "Adios, " + msg;
    }

}
```

- Via programmatically:

```
isCallerInRole(String role): Returns a boolean indicating whether a user belong to a given role.

getCallerPrincipal(): Returns the currently authenticated user.
```

Example:

```java
@Stateless
public class HelloWorldEJB implements HW {
    @Resource
    EJBContext context;

    public String HelloWorld() {
        if (context.isCallerInRole("admin")) {
          return "Hello " + context.getCallerPrincipal().getName();
        } else {
          return "Unauthorized user.";
        }
    }
}
```

- Via RESTEasy Annotations: @PermitAll, @RolesAllowed("admin, guest"), @DenyAll

- Security Realms in EAP

By default, EAP defines the ApplicationRealm, which uses the following files to store the users and their roles, respectively:

```
application-users.properties
application-roles.properties
```

Although more security realms can be added to EAP, this course primarily uses the ApplicationRealm for authentication and authorization. The following is the default configuration for the ApplicationRealm:

```xml
  <security-realm name="ApplicationRealm">
                <authentication>
                    <local default-user="$local" allowed-users="*" skip-group-loading="true"/>
                    <properties path="application-users.properties" relative-to="jboss.server.config.dir"/>
                </authentication>
                <authorization>
                    <properties path="application-roles.properties" relative-to="jboss.server.config.dir"/>
                </authorization>
  </security-realm>
```

Notice that the realm has a tag each for <authentication> and <authorization>. The <authentication> tag defines the path to the user properties file. In this case, that file is application-users.properties in the EAP server configuration directory. This file stores user names and passwords as a key-value pair, for example:

```
<username>=<password>
```

The <authorization> tag defines the path to the roles properties file. In this case, that file is application-roles.properties, which is located in the EAP server configuration directory. This file stores users and roles as key-value pairs using the following syntax:

```
<role>=<user1>,<user2>...
```

- Login Modules: by files, SQL, etc.

While the security realms in EAP are used to configure the user credentials, a security domain defines a set of JAAS declarative security configurations. The biggest advantage of using a declarative approach to security is that it separates a lot of the security details from the application itself. This provides additional flexibility and keeps the application's business logic code readable and easier to maintain by removing the implementation details for the security technology that the application server uses.

EAP includes several built-in login modules that developers can use for authentication within a security domain. These login modules include the ability to read user information from a relational database, an LDAP server, or flat files. It is also possible to build a custom module depending on the security requirements of the application.

The method for user authentication is defined in the security domain. By default, EAP defines the other security domain with the following configuration:

```xml
<security-domain name="other" cache-type="default">
    <authentication>
        ...
        <login-module code="RealmDirect" flag="required">
            <module-option name="password-stacking" value="useFirstPass"/>
        </login-module>
    </authentication>
</security-domain>
```

The RealmDirect login module uses an application realm when making authentication and authorization decisions. If no realm is specified, then the module uses the ApplicationRealm, and therefore the users and roles property files for authentication and authorization.

To enable the other security domain in the application, add the following tag to the application's src/main/webapp/WEB-INF/jboss-web.xml:

```xml
<security-domain>other</security-domain>
```

Finally, to complete authentication, configure the WEB-INF/web.xml file in the application to define the <login-config>. The following example defines the <login-config> to use BASIC authentication with the ApplicationRealm. With this configuration in place, users are prompted for credentials when they access a server resource and the server verifies the credentials against the ApplicationRealm.

```xml
 <login-config>
      <auth-method>BASIC</auth-method>
      <realm-name>ApplicationRealm</realm-name>
   </login-config>
</web-app>
```
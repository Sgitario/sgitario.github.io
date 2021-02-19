---
layout: post
title: Blockchain using Custom Ethereum Network in Quarkus
date: 2020-07-10
tags: [ Blockchain, Quarkus ]
---

We already introduced Blockchain and Ethereum in [an earlier post](https://sgitario.github.io/ethereum-and-solidity-getting-started/) and also introduced a simple Lottery example [in Spring Boot](https://sgitario.github.io/blockchain-using-springboot/). 

In this post, we'll implement the same Lottery example in [Quarkus](https://quarkus.io/), the most promising Java framework at the moment. 
Quarkus is getting very popular because it's really fast and requires less memory than a Spring application does. So, what if we could implement a Quarkus application to mine an Ethereum network? We could deploy **MUCH MORE** applications to mine using the same resources that using a Spring application. 

What do we need? Because the purpose of this blog is learning. We're going to:
- Create a custom Web3j Client extension for Quarkus: so everybody could reuse this for any purposes
- Create a Lottery application in Quarkus using the custom extension and Solidity as we did [in Spring Boot](https://sgitario.github.io/blockchain-using-springboot/)
- Add Integration Tests for covering all the functionality in the application. 

## Create Custom Web3j Client Extension

Extensions in Quarkus are used for providing some common functionality across any Quarkus application. What functionality do we want to expose here? Just to configure the Web3j Client for us. So, by adding this extension in my Quarkus application, I would expect to be able to inject the client like this:

```java
@Inject
Web3j web3j;
``` 

Then, we can configure this instance by properties. Simple like that. 

Next, we need to understand about how Quarkus works. Quarkus does a lot in build time and that's why it's so fast. When we run the application, many things are already done. Getting back to our web3j client extension, at build time, we want to declare the Web3j service to be exposed and also to be able to configure this service at runtime, so I can choose what configuration to use without needing to compile the whole app.

There are a lot of [guides](https://quarkus.io/guides/writing-extensions) about how to create a custom extension in Quarkus, so we will not give many details.

For start, we need to create the custom extension using maven command line:

``` 
mvn io.quarkus:quarkus-maven-plugin:1.6.0.Final:create-extension -N \
    -Dquarkus.artifactIdBase=web3j-client \
    -Dquarkus.artifactIdPrefix=quarkus- \
    -Dquarkus.nameBase="Web3j Client Extension"
``` 

This will create the parent module and two subfolders:

- runtime: to expose all what the apps can use
- deployment: to instrumentalize the build in Quarkus

### Runtime

As I said, our objective is to expose the Web3j client, so we need the _Web3jClientProducer.java_:

```java
@ApplicationScoped
public class Web3jClientProducer {

	private volatile Web3jConfiguration config;

	void initialize(Web3jConfiguration config) {
		this.config = config;
	}

	@Singleton
	@Produces
	public Web3j web3j() {
		return Web3j.build(new HttpService(config.url));
	}
}
```

I guess the CDI annotations are already familiar for you, so I won't explain them. The important bit here is that the producer expects the _Web3jConfiguration.java_:

```java
@ConfigRoot(phase = ConfigPhase.RUN_TIME)
public class Web3jConfiguration {
	/**
	 * Web3j URL to use.
	 */
	@ConfigItem
	public String url;
}
```

The configuration only has one parameter, the ethereum URL to connect with. Note that we are saying that this configuration might change at runtime. These annotations are used by Quarkus to check whether everything is configured properly.

We're still missing who is injecting the configuration to the producer. This is done by a component called in Quarkus: Recorder. _Web3jRecorder.java_:

```java
@Recorder
public class Web3jRecorder {

	public void update(BeanContainer beanContainer, Web3jConfiguration configuration) {
		Web3jClientProducer producer = beanContainer.instance(Web3jClientProducer.class);
		producer.initialize(configuration);
	}

}
```

Again, the **@Recorder** annotation is used by Quarkus to check everything is configured ok. 

### Deployment

We already have all our components and the service to be exposed, but who is calling the recorder? This is done in the deployment module by a component called in Quarkus **Procesors**. See _QuarkusWeb3jClientProcessor.java_:

```java
class QuarkusWeb3jClientProcessor {

	private static final String FEATURE = "web3j-client";

	@BuildStep
	void build(BuildProducer<FeatureBuildItem> featureProducer,
			BuildProducer<AdditionalBeanBuildItem> additionalBeanProducer,
			BuildProducer<BeanContainerListenerBuildItem> containerListenerProducer) {

		featureProducer.produce(new FeatureBuildItem(FEATURE));

		additionalBeanProducer.produce(AdditionalBeanBuildItem.unremovableOf(Web3jClientProducer.class));
	}

	@BuildStep
	@Record(ExecutionTime.RUNTIME_INIT)
	void configureProducer(Web3jRecorder recorder, BeanContainerBuildItem beanContainerBuildItem,
			Web3jConfiguration configuration) {
		recorder.update(beanContainerBuildItem.getValue(), configuration);
	}

}
```

We're saying that the extension is exposing a feature called "web3j-client" and that the configuration must be done at runtime.

How can we be sure that everything is working as expected? Let's add an integration test in the deployment module:

```java
public class QuarkusWeb3jClientTest {
	@RegisterExtension
	static final QuarkusUnitTest config = new QuarkusUnitTest()
			.setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class))
			.withConfigurationResource("application.properties");

	@Inject
	Web3jConfiguration web3j;

	@Test
	public void checkUrlIsLoaded() {
		assertEquals(web3j.url, "http://localhost:8545");
	}
}
```

Note that in order to use the QuarkusUnitTest, we need to add this dependency in the deployment maven pom file:

```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-junit5-internal</artifactId>
  <scope>test</scope>
</dependency>
```

## Lottery Application

A good starting point for our Quarkus application, it's to use the [https://code.quarkus.io/](https://code.quarkus.io/) site. For this example, I selected the resteasy-jsonb and smallrye-openapi extensions.

Let's now use our custom extension to create the Quarkus application. For this, we only need to add the maven dependency in our _pom.xml_ file:

```xml
<dependency>
  <groupId>org.sgitario.quarkus</groupId>
  <artifactId>quarkus-web3j-client</artifactId>
</dependency>
```

Next, we're going to implement the same service as done [in Spring Boot](https://sgitario.github.io/blockchain-using-springboot/):

```java
@ApplicationScoped
public class LotteryService {

	private static final Logger LOGGER = Logger.getLogger(LotteryService.class);

	@Inject
	Web3j web3j;
	@Inject
	LotteryProperties config;

	private Map<String, String> lotteryByOwner = new ConcurrentHashMap<>();

	public BigDecimal getBalance(String owner) throws IOException {
		BigInteger wei = web3j.ethGetBalance(lotteryByOwner.get(owner), DefaultBlockParameterName.LATEST).send()
				.getBalance();
		return Convert.fromWei(wei.toString(), Unit.ETHER);
	}

	public void join(String owner, String account, BigDecimal ethers) throws Exception {
		Lottery lottery = loadContract(owner, account);
		TransactionReceipt tx = lottery.enter(Convert.toWei(ethers, Unit.ETHER).toBigInteger()).send();
		tx.getLogs().forEach(LOGGER::info);
	}

	public String deployLottery(String owner) throws Exception {
		LOGGER.info("Deploying Lottery...");
		Lottery contract = Lottery.deploy(web3j, txManager(owner), config.gas()).send();
		LOGGER.info("Deployed new contract with address: " + contract.getContractAddress());
		lotteryByOwner.put(owner, contract.getContractAddress());
		return contract.getContractAddress();
	}

	@SuppressWarnings("unchecked")
	public List<String> getPlayers(String owner) throws Exception {
		Lottery lottery = loadContractFromOwner(owner);
		List<String> players = lottery.getPlayers().send();
		return players;
	}

	public String pickWinner(String owner) throws Exception {
		Lottery lottery = loadContractFromOwner(owner);
		lottery.pickWinner().send();
		lotteryByOwner.remove(owner);
		return lottery.getWinner().send();
	}

	private Lottery loadContractFromOwner(String owner) {
		return loadContract(owner, owner);
	}

	private Lottery loadContract(String owner, String accountAddress) {
		return Lottery.load(lotteryByOwner.get(owner), web3j, txManager(accountAddress), config.gas());
	}

	private TransactionManager txManager(String address) {
		return new ClientTransactionManager(web3j, address);
	}
}
```

This service is using some properties that belong only to our Lottery application:

```java
@ConfigProperties(prefix = "lottery.contract")
public class LotteryProperties {
	private BigInteger gasPrice;
	private BigInteger gasLimit;

	// getters and setters

	@Transient
	public StaticGasProvider gas() {
		return new StaticGasProvider(gasPrice, gasLimit);
	}
}
```

And again we're going to expose this service via REST:

```java
@Path("/lottery")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@Tags(value = @Tag(name = "lottery", description = "All the Lottery methods"))
public class OwnerResource {

	@Inject
	LotteryService service;

	@POST
	@Path("/{owner}")
	@Operation(summary = "Deploy a new Lottery ")
	@APIResponses(value = { @APIResponse(responseCode = "200", description = "Return the lottery account") })
	public Account deploy(@Parameter(description = "Owner account", required = true) @PathParam("owner") String owner)
			throws Exception {
		Account account = new Account();
		account.setAccount(service.deployLottery(owner));
		return account;
	}

	@GET
	@Path("/{owner}/balance")
	@Operation(summary = "Get the balance of the Lottery contract")
	@APIResponses(value = { @APIResponse(responseCode = "200", description = "Return the lottery balance") })
	public Quantity getBalance(
			@Parameter(description = "Owner account", required = true) @PathParam("owner") String owner)
			throws Exception {
		Quantity quantity = new Quantity();
		quantity.setEthers(service.getBalance(owner));
		return quantity;
	}

	@POST
	@Path("/{owner}/finish")
	@Operation(summary = "Finish the Lottery and pick a winner")
	@APIResponses(value = {
			@APIResponse(responseCode = "200", description = "Return the winner account", content = @Content(schema = @Schema(implementation = Account.class))) })
	public Account pickWinner(
			@Parameter(description = "Owner account", required = true) @PathParam("owner") String owner)
			throws Exception {
		Account winner = new Account();
		winner.setAccount(service.pickWinner(owner));
		return winner;
	}

	@GET
	@Path("/{owner}/players")
	@Operation(summary = "Return all the players in the lottery")
	@APIResponses(value = { @APIResponse(responseCode = "200", description = "Return the list of players") })
	public List<String> getPlayers(
			@Parameter(description = "Owner account", required = true) @PathParam("owner") String owner)
			throws Exception {
		return service.getPlayers(owner);
	}

	@POST
	@Path("/{owner}/players/{account}")
	@Consumes(MediaType.APPLICATION_JSON)
	@Operation(summary = "Join a new player in the lottery")
	public void joinLottery(@Parameter(description = "Owner account", required = true) @PathParam("owner") String owner,
			@Parameter(description = "Player account", required = true) @PathParam("account") String account,
			@Parameter(description = "Quantity", required = true) Quantity quantity) throws Exception {
		service.join(owner, account, quantity.getEthers());
	}
}
```

Note that I'm using [the OpenAPI annotations](https://quarkus.io/guides/openapi-swaggerui) to document my REST API. 

### Integration Tests

For covering this functionality via integration tests, we need an Ethereum network running and some accounts. For this, I'm going to use the [testcontainers](https://www.testcontainers.org/) framework which uses Docker to startup containers and works really well with [JUnit 5](https://junit.org/junit5/docs/current/user-guide/). 

```xml
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>testcontainers</artifactId>
  <scope>test</scope>
</dependency>
```

For running the Ethereum network, we're going to use [the Ganache docker](https://hub.docker.com/r/trufflesuite/ganache-cli/) image:

```java
public class GanacheTestResource implements QuarkusTestResourceLifecycleManager {

	public static final String URL_PROPERTY = "quarkus.web3j.url";
	private static final int PORT = 8545;
	private static final Logger LOGGER = LoggerFactory.getLogger(GanacheTestResource.class);

	private final GenericContainer<?> resource;

	@SuppressWarnings("resource")
	public GanacheTestResource() {
    resource = new GenericContainer<>("trufflesuite/ganache-cli")
        .withExposedPorts(PORT)
        .withCommand("--accounts 10")
        .withLogConsumer(new Slf4jLogConsumer(LOGGER))
				.waitingFor(Wait.forLogMessage(".*Listening on 0.0.0.0.*", 1));
	}

	@Override
	public Map<String, String> start() {
		resource.start();
		return Collections.singletonMap(URL_PROPERTY, getNodeUrl());
	}

	@Override
	public void stop() {
		resource.stop();
	}

	private String getNodeUrl() {
		return String.format("http://localhost:%s", resource.getMappedPort(PORT));
	}

}
```

The interface **QuarkusTestResourceLifecycleManager** is used to integrate this test resource with the Quarkus test lifecycle (to inject properties as an example).

And finally, our integration tests:

```java
@QuarkusTest
@QuarkusTestResource(GanacheTestResource.class)
public class OwnerResourceTest {

	private static final String OWNER_PATH = "/lottery/%s";

	@Inject
	Web3j web3j;

	private String owner;
	private Iterator<String> availableAccounts;
	private Response response;

	@BeforeEach
	public void setup() throws IOException {
		RestAssured.defaultParser = Parser.JSON;
		List<String> accounts = web3j.ethAccounts().send().getAccounts();
		owner = accounts.get(0);
		List<String> players = new ArrayList<>(accounts.size() - 1);
		for (int index = 1; index < accounts.size(); index++) {
			players.add(accounts.get(index));
		}

		availableAccounts = players.iterator();
	}

	@Test
	public void testDeploy() {
		whenDeployLottery();
		thenReturnsOk();
	}

	@Test
	public void testNoPlayers() {
		givenLottery();
		whenGetPlayers();
		thenPlayersIsEmpty();
	}

	@Test
	public void testJoinPlayers() {
		givenLottery();
		String newPlayer = whenJoinNewPlayer(2.0);
		whenGetPlayers();
		thenPlayerIsFound(newPlayer);
	}

	@Test
	public void testBalance() {
		givenLottery();
		givenPlayerInLotteryWithEther(2.0);
		whenGetBalance();
		thenBalanceIs(2.0);
	}

	@Test
	public void testPickWinner() {
		givenLottery();
		String player = givenPlayerInLotteryWithEther(2.0);
		whenPickWinner();
		thenWinnerIs(player);
	}

	// given, when, then methods

}
```

| We didn't include the given, when, then methods on purpose. Refer to [my GitHub repository](https://github.com/Sgitario/lottery-web3j-quarkus) for this.

## Demostration

Let's play a bit with this Lottery application. First, we need to startup an Ethereum network to work with:

```
docker run --name ganache -p 8545:8545 trufflesuite/ganache-cli:v6.10.0-beta.2 --accounts 2
```

We should see the output:

```
Available Accounts
==================
(0) 0x1d8ABF1CC1CB2c38F97489805BBD8Ad577B6aD1b (100 ETH)
(1) 0xC51dA14cFb59a91413EaDf08B6f1F8157032e2ad (100 ETH)
...

Listening on 0.0.0.0:8545
```

This is the owner account we're going to use **0x1d8ABF1CC1CB2c38F97489805BBD8Ad577B6aD1b** and the player account **0xC51dA14cFb59a91413EaDf08B6f1F8157032e2ad**.

Now, we're going to launch our application in DEV mode:

```
mvn -Dquarkus.web3j.url=http://localhost:8545 compile quarkus:dev
```

Or we can build our application in native mode:

```
mvn clean package -Pnative
./target/lottery-web3j-app/target/lottery-web3j-app-1.0-SNAPSHOT-runner
```

You should see the output:

```
Listening for transport dt_socket at address: 5005
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2020-07-10 13:02:48,641 INFO  [io.quarkus] (Quarkus Main Thread) lottery-web3j-app 1.0-SNAPSHOT on JVM (powered by Quarkus 1.6.0.Final) started in 0.994s. Listening on: http://0.0.0.0:8080
2020-07-10 13:02:48,644 INFO  [io.quarkus] (Quarkus Main Thread) Profile dev activated. Live Coding activated.
2020-07-10 13:02:48,644 INFO  [io.quarkus] (Quarkus Main Thread) Installed features: [cdi, resteasy, resteasy-jsonb, smallrye-openapi, swagger-ui, web3j-client]
```

| We can browse to http://localhost:8080/swagger-ui to try the REST API out.

- Deploy a Lottery contract:

```
curl -X POST "http://localhost:8080/lottery/0x1d8ABF1CC1CB2c38F97489805BBD8Ad577B6aD1b" -H "accept: application/json" -d ""
```

- Get balance:

```
curl -X GET "http://localhost:8080/lottery/0x1d8ABF1CC1CB2c38F97489805BBD8Ad577B6aD1b/balance" -H "Content-Type: application/json"
```

- Get players:

```
curl -X GET "http://localhost:8080/lottery/0x1d8ABF1CC1CB2c38F97489805BBD8Ad577B6aD1b/players" -H "Content-Type: application/json"
```

- Join Player:

```
curl -X POST "http://localhost:8080/lottery/0x1d8ABF1CC1CB2c38F97489805BBD8Ad577B6aD1b/players/0xC51dA14cFb59a91413EaDf08B6f1F8157032e2ad" -H "Content-Type: application/json" -d "{\"ethers\":2}"
```

- Pick a winner balance:

```
curl -X POST "http://localhost:8080/lottery/0x1d8ABF1CC1CB2c38F97489805BBD8Ad577B6aD1b/finish" -H "accept: application/json" -d ""
```

## Conclusion

I hope you have enjoyed this post as much I do!

The source code of this example can be found [here](https://github.com/Sgitario/lottery-web3j-quarkus).
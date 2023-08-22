+++
title = "Websockets"
description = ""
tags = [
    "spring",
    "websockets",
    "mongodb",
]
date = "2014-04-02"
categories = [
    "software",
    "engineering",
]
draft = false
+++

Today post is about a small experiment using [Spring 5](https://spring.io/) to play with [WebFux](https://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/html/web-reactive.html), I've used it to create a small WebSocket controller to "simulate" an e-mail inbox. The idea was to send some dummy text to the backend, persist it on [MongoDB](https://www.mongodb.com/) and from time to time check for new messages and send it to the clients connected to the WebSocket endpoint.

## The Cast

Let's see an overview of all actors involved in this small experiment. This text probably will be much longer than the code necessary to have everything working with Spring 5. 

I'll assume that Spring doesn't need any introduction even because is hard to define it nowadays, if we were back to 2002 ish I would say it is an Ioc container blah blah blah but today...


### [MongoDB](https://www.mongodb.com/)

> MongoDB is a free and open-source cross-platform document-oriented database program. Classified as a NoSQL database program, MongoDB uses JSON-like documents with schemata.

### [MongoDB Reactive Streams Java Driver](https://mongodb.github.io/mongo-java-driver-reactivestreams/)

> MongoDB Reactive Streams Java Driver, providing asynchronous stream processing with non-blocking back pressure for MongoDB.

### [Project Reactor](https://projectreactor.io/)

> Non-Blocking Reactive Streams Foundation for the JVM both implementing a Reactive Extensions inspired API and efficient event streaming support.

> The reactive design pattern is an event-based architecture for asynchronous handling of a large volume of concurrent service requests coming from single or multiple service handlers.

> And the Spring Reactor project is based on this pattern and has the clear and ambitious goal of building asynchronous, reactive applications on the JVM.

### [WebFlux](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html)

> Spring WebFlux is an asynchronous framework from the bottom up. It can run on Servlet Containers using the Servlet 3.1 non-blocking IO API as well as other async runtime environments such as netty or undertow.

### [WebSocket](https://en.wikipedia.org/wiki/WebSocket)

> WebSocket is a computer communications protocol, providing full-duplex communication channels over a single TCP connection.
> 
> The WebSocket protocol enables interaction between a web client (such as a browser) and a web server with lower overheads, facilitating real-time data transfer from and to the server. This is made possible by providing a standardized way for the server to send content to the client without being first requested by the client, and allowing messages to be passed back and forth while keeping the connection open. In this way, an ongoing two-way conversation can take place between the client and the server.

## The actual code

### pom.xml

Let's start with the [pom.xml](https://github.com/allandequeiroz/java/blob/spring.reactive-delayed-messages/pom.xml) file, this way you'll have what it takes to play by yourself with your WebSockets. Let's check the most important parts.

The parent pom or the BOM file used to predefine the dependencies versions, so you can add your dependencies without having to figure out about version compatibility between the components.

> BOM stands for Bill Of Materials. A BOM is a particular kind of POM that is used to control the versions of a project’s dependencies and provide a central place to define and update those versions.

> BOM provides the flexibility to add a dependency to our module without worrying about the version that we should depend on.

<pre class="line-numbers language-xml">
<code>&lt;parent&gt;
    &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
    &lt;artifactId&gt;spring-boot-starter-parent&lt;/artifactId&gt;
    &lt;version&gt;2.0.4.RELEASE&lt;/version&gt;
&lt;/parent&gt;
</code></pre>

The next part, the dependencies I've used, observe that except for the most specific dependencies I didn't configure any version, it's all done by the parent pom.

<pre class="line-numbers language-xml">
<code>&lt;dependencies&gt;
    &lt;!-- Boot --&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-starter-websocket&lt;/artifactId&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-starter-actuator&lt;/artifactId&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-starter-data-mongodb-reactive&lt;/artifactId&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-starter-webflux&lt;/artifactId&gt;
    &lt;/dependency&gt;

    &lt;!-- Tooling --&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.projectlombok&lt;/groupId&gt;
        &lt;artifactId&gt;lombok&lt;/artifactId&gt;
        &lt;optional&gt;true&lt;/optional&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-devtools&lt;/artifactId&gt;
        &lt;scope&gt;runtime&lt;/scope&gt;
    &lt;/dependency&gt;

    &lt;!-- Websocket --&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.webjars&lt;/groupId&gt;
        &lt;artifactId&gt;webjars-locator-core&lt;/artifactId&gt;
    &lt;/dependency&gt;

    &lt;!-- Test --&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-starter-test&lt;/artifactId&gt;
        &lt;scope&gt;test&lt;/scope&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;io.projectreactor&lt;/groupId&gt;
        &lt;artifactId&gt;reactor-test&lt;/artifactId&gt;
        &lt;scope&gt;test&lt;/scope&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;commons-lang&lt;/groupId&gt;
        &lt;artifactId&gt;commons-lang&lt;/artifactId&gt;
        &lt;version&gt;2.6&lt;/version&gt;
        &lt;scope&gt;test&lt;/scope&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.easytesting&lt;/groupId&gt;
        &lt;artifactId&gt;fest-assert&lt;/artifactId&gt;
        &lt;version&gt;1.4&lt;/version&gt;
        &lt;scope&gt;test&lt;/scope&gt;
    &lt;/dependency&gt;
&lt;/dependencies&gt;
</code></pre>

I won't add the whole pom here. If you want, you can generate your one, indeed more updated from the [SPRING INITIALIZR](https://start.spring.io/) page. Pick any functionality you need for your project, and it will just generate a skeleton for you.

### application.properties

This one is a nice to have, especially if you're going to use database connections, you can place your configurations here instead of having them hardcoded and spread through your code. In case you need extra configuration, and you don't know for sure what property you need, you can check the [Common application properties](https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html).

For the today's experiment, what I needed was just some port definition, logging, and MongoDB URI connection.

<pre class="line-numbers language-properties">
<code>server.port=8080

#logging
logging.level.org.springframework.data=debug
logging.level.=debug

#mongodb
spring.data.mongodb.uri=mongodb://r640:27017/chat
</code></pre>

### WebSocket configurations

The code below is all you need to have your initial WebSocket configuration, I saw some more complex code essentially doing the same thing, but I suppose they were from previous versions of Spring, with Spring 5, that's all you need to start.

<pre class="line-numbers language-java">
<code>@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

   @Override
   public void configureMessageBroker(MessageBrokerRegistry config) {
      config.enableSimpleBroker("/topic");
      config.setApplicationDestinationPrefixes("/app");
   }

   @Override
   public void registerStompEndpoints(StompEndpointRegistry registry) {
      registry.addEndpoint("/email");
      registry.addEndpoint("/email").withSockJS();
      registry.addEndpoint("/emails");
      registry.addEndpoint("/emails").withSockJS();
   }
}
</code></pre>

The interesting points here are first the `@EnableWebSocketMessageBroker` annotation, here we're telling Spring that you want to provide WebSockets capabilities on your application, `config.enableSimpleBroker("/topic")` tells spring that anything starting with `/topic` is a WebSocket endpoint where clients can subscribe to, `config.setApplicationDestinationPrefixes("/app")` again gives Spring some information, now it tells that clients will send data to your application through any endpoint starting with `/app`, `registry.addEndpoint("/email")` here we're telling Spring about the endpoints we want to use the STOMP protocol, notice that we're overriding `registerStompEndpoints` these endpoints will probably match some `@MessageMapping` on your controller.

### Model

Here we're primarily providing the definitions of how the MongoDB collection will look like, notice `@Document` and `@Id` annotations, they are basically saying to Spring that this class is about collection entries called `email` identified by the `id` field, if we want the collection to have a different name we could define Document like `@Document(collection = "another_name")`

<pre class="line-numbers language-java">
<code>@Data
@Builder
@Document
public class Email {
   @Id
   private String id;
   private String content;
   private final Date sentAt = new Date();
}
</code></pre>

### Repository

This part is surprising if there's no need for anything special from the repository, the only thing needed is an interface extending `ReactiveMongoRepository`, Spring will provide basic operations out of the box.

To get just the newest emails since the last verification all we need is to provide an abstract method with a `@Query` annotation, and anything else is handled by Spring, what a wonderful world we're living these days :)

<pre class="line-numbers language-java">
<code>@Repository
public interface EmailRepository extends ReactiveMongoRepository<Email, String> {
   @Query("{ 'sentAt' : { $gte : ?0 }}")
   Flux<Email> findLastOnes(Date lastExecution);
}
</code></pre>

### The controller

Please ignore the `private Date lastExecution = new Date()` over there, by default Spring creates singleton beans so we're safe here and please, stop thinking about a service layer if you're doing so, we don't need it here right now.

<pre class="line-numbers language-java">
<code>@Controller
@EnableScheduling
public class EmailController {

   @Autowired
   private EmailRepository repository;

   @Autowired
   private SimpMessagingTemplate template;

   private Date lastExecution = new Date();

   @MessageMapping("/email")
   public void email(final Email message) {
      repository.save(message).block();
   }

   @MessageMapping("/emails")
   public void emails() {
      repository.findAll().subscribe(new EmailSubscriber<>());
   }

   @Scheduled(fixedRate = 10000)
   public void updateClients() {
      repository.findLastOnes(lastExecution).subscribe(new EmailSubscriber<>());
   }

   private class EmailSubscriber<T> extends BaseSubscriber<T> {
      @Override
      protected void hookOnComplete() {
         lastExecution = new Date();
      }

      @Override
      protected void hookOnError(Throwable throwable) {
         template.convertAndSend("/topic/email/errors", throwable.getMessage());
      }

      @Override
      protected void hookOnNext(T value) {
         final Email email = (Email) value;
         email.setContent(HtmlUtils.htmlEscape(email.getContent()));
         template.convertAndSend("/topic/email/updates", email);
      }
   }
}
</code></pre>

Now, the interesting parts here, the `@MessageMapping` annotated methods are the ones that clients will send requests too, just remember that from the client side, they need to hit for example `/app/email` instead of just `/email`, do you remember `WebSocketConfig`, `config.setApplicationDestinationPrefixes("/app")`?

To simulate that small delay from the point when someone sends us an email and the moment we receive it, I've configured an `schedule` that runs every 10 seconds, take a look at `updateClients`. Here some interesting things are going on. First, we're asking the repository to give us the new messages since the last verification and, instead of wait for the answer, we've subscribed, furthermore when something comes back we'll be notified so we can push content to the connected clients when and just when we have something to do so.

`subsctibe` is a method provided by [Flux](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html) that in the Reactor world represents a Reactive stream of size from 0 to N, for 0 or single result [Mono](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html) is the right representation. Ok, we have some results, what happens now? The Subscriber `hookOnNext` method will be executed once for each of the found results. Here we can do some work like escape characters. Because we need at least one method with more than one line right? From this point just sent it to the proper `topic` and the subscribed clients will be updated.

### Testing

Testing it was simple too since with a simple `@SpringBootTest` annotation the whole stack was made available, we only have to do some initial setup to have an integration test up and running.

<pre class="line-numbers language-java">
<code>@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class EmailControllerTest {

   private static final String SUBSCRIBE_TOPIC_ENDPOINT = "/topic/email/updates";
   private static final String SEND_EMAIL_ENDPOINT = "/app/email";

   @Autowired
   private EmailRepository repository;

   @Autowired
   private ReactiveCrudRepository operations;

   @Value("${local.server.port}")
   private int port;
   private String URL;

   private CompletableFuture<Email> completableFuture;
   private WebSocketStompClient stompClient;
   private StompSession stompSession;

   @Before
   public void setup() throws InterruptedException, ExecutionException, TimeoutException {
      completableFuture = new CompletableFuture<>();
      URL = String.format("ws://localhost:%d/email", port);

      stompClient = new WebSocketStompClient(new SockJsClient(createTransportClient()));
      stompClient.setMessageConverter(new MappingJackson2MessageConverter());
      stompSession = stompClient.connect(URL, new StompSessionHandlerAdapter() {
      }).get(1, TimeUnit.SECONDS);
   }

   @Test
   public void receiveEmails() throws InterruptedException, ExecutionException, TimeoutException {
      // Adding a dummy email
      final Email email = Email
            .builder()
            .id(UUID.randomUUID().toString())
            .content(RandomStringUtils.randomAlphabetic(10))
            .build();

      // Send the email through webSocket to be persisted
      stompSession.send(SEND_EMAIL_ENDPOINT, email);

      // Subscribing to the notification endpoint as a client
      stompSession.subscribe(SUBSCRIBE_TOPIC_ENDPOINT, new MessageStompFrameHandler());

      // Waiting up to 10s for a notification to match the scheduler
      final Email emailReceived = completableFuture.get(10, TimeUnit.SECONDS);

      // Validating content
      assertThat(emailReceived).isNotNull();
      assertThat(emailReceived.getId()).isEqualTo(email.getId());
      assertThat(emailReceived.getContent()).isEqualTo(email.getContent());
      assertThat(emailReceived.getSentAt()).isEqualTo(email.getSentAt());

      // Removing dummy
      operations.delete(email).block();

      // Verifying if is really gone
      final Email foundEmail = repository.findById(email.getId()).block();
      assertThat(foundEmail).isNull();
   }

   private List<Transport> createTransportClient() {
      final List<Transport> transports = new ArrayList<>(1);
      transports.add(new WebSocketTransport(new StandardWebSocketClient()));
      return transports;
   }

   private class MessageStompFrameHandler implements StompFrameHandler {

      @Override
      public Type getPayloadType(StompHeaders stompHeaders) {
         return Email.class;
      }

      @Override
      public void handleFrame(StompHeaders stompHeaders, Object o) {
         completableFuture.complete((Email) o);
      }
   }
}
</code></pre>

### The client

To play here I've wrote a `html`, `js` and `css` files, placed them inside the `src/main/resources/static` folder and started the project, when hitting `localhost:8080` the client gets connected automatically, and the fun is complete, I'm not proud of them, but you can have them [index.html](https://gist.github.com/allandequeiroz/7273457217dfa02a36c40728a93fccec), [index.js](https://gist.github.com/allandequeiroz/a9740820d2a346c9b5d5c6b2f070b79f) and [index.css](https://gist.github.com/allandequeiroz/3dfa5eb4ee4870507b5f73cb6b607762).

## Conclusion

Once again Spring did an excellent job to provide ways to write applications with low effort. With few lines of code, we can see a new paradigm acting over every single layer of our application. And in case you're wondering if it could be applied somewhere else but WebSockets related endpoints, you can use mainly the same dependencies and approach to see similar results happening for example for [rest endpoints](https://github.com/allandequeiroz/java/tree/spring.reactive-rest-endpoint).

Hope it provided you a glance of what's possible to achieve with this new Spring version and how much fun you can have playing around with these new toys :)

## References

[Spring Boot](https://spring.io/projects/spring-boot)

[Spring 5](https://spring.io/)

[WebFux](https://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/html/web-reactive.html)

[MongoDB](https://www.mongodb.com/)

[WebSocket](https://en.wikipedia.org/wiki/WebSocket)

[WebSocket Support on Spring 5](https://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/html/websocket.html)

[Stomp](https://stomp.github.io/)

[Project Reactor](https://projectreactor.io/docs/core/release/reference/docs/index.html)

[Reactive Streams](http://www.reactive-streams.org/)

[Spring Initializr](https://start.spring.io/)

[Common Spring application properties](https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html)

---
layout: post
title: Simple REST server written in native Java
date: 2024-08-07
tags: [ Java ]
---

Sometimes it's good to see how things work or look like without using a framework that hide/ease these things. That's why I wanted to write a simple REST service in Java following the next objectives:
- Using no framework at all
- Using TDD, clean code and SOLID patterns
- Taking into account the concurrent users that it can support.

Moreover, I won't be using any data layer at all, but will be using an in-memory storage instead.

# Starting point

I will be using Maven to build my solution and the following test dependencies to write test scenarios:
- junit 5: which is almost the standard for writing unit tests in Java.
- rest-assured: to test REST interfaces easily

Note that you can download the source code in [here](https://github.com/Sgitario/rest-server-java).

Where the main class `org.sgitario.Application` is:

```java
package org.sgitario;

public class Application {
    public static void main(String[] args) {
        System.out.println("Hello World!");
    }
}
```

When building the application using `mvn clean install` and then running the jar file using `java -jar target/pure-java-rest-api-1.0-SNAPSHOT.jar`, we will see `Hello World!`.

# 1: Functional Requirements

What do we want to design?

- GET method `/api/books` that returns the list of books we have in our store.
- PUT method `/api/books/{title}` that adds a new book with its author. 
- GET method `/api/books/{title}` that returns the author of the book
- DELETE method `/api/books/{title}` that deletes an existing book. 
- POST method `/api/books` that creates/updates the book's author. 

# 2: Implement GET method `/api/books`

Let's first write the test:

```java
class GetBooksIntegrationTest extends BaseIntegrationTest {

    @Test
    void testGetBooksWhenNoBooksThenItShouldReturnEmptyList() {
        get("/api/books")
                .then()
                .statusCode(200)
                .body(is("[]"));
    }
}
```

Update the application to start our server:

```java
public class Application {

    public static final int SERVER_PORT = 8000;

    private HttpServer server;

    public void start() throws IOException {
        var controllers = ServiceLoader.load(RestController.class);
        server = HttpServer.create(new InetSocketAddress(SERVER_PORT), 0);
        server.createContext("/", (exchange -> {
            boolean handled = false;
            for (var controller : controllers) {
                if (exchange.getRequestURI().getPath().matches(controller.path())
                        && exchange.getRequestMethod().equalsIgnoreCase(controller.method().name())) {
                    String response = controller.handle(exchange);
                    if (response != null) {
                        OutputStream output = exchange.getResponseBody();
                        output.write(response.getBytes());
                        output.flush();
                    }

                    exchange.close();
                    handled = true;
                    break;
                }
            }

            if (!handled) {
                exchange.sendResponseHeaders(404, -1);
            }
        }));

        server.setExecutor(Executors.newCachedThreadPool()); // creates a default executor
        server.start();
    }

    public void stop() {
        if (server != null) {
            server.stop(0);
            server = null;
        }
    }

    public static void main(String[] args) throws IOException {
        new Application().start();
    }
}
```

Notes:
- We have used the ServiceLoader API from Java, so we can easily add/modify the REST controllers we'll support
- We have implemented the start/stop methods, so we can easily start and stop the server in the tests

Next, the controller:

```java
public class GetBooksRestController implements RestController {

    private final BookService bookService;

    public GetBooksRestController() {
        this(new BookService());
    }

    public GetBooksRestController(BookService bookService) {
        this.bookService = bookService;
    }

    @Override
    public String path() {
        return "/api/books$";
    }

    @Override
    public Method method() {
        return Method.GET;
    }

    @Override
    public String handle(HttpExchange exchange) throws IOException {
        String response = bookService.getBooks().stream().map(Book::title).toList().toString();
        exchange.sendResponseHeaders(200, response.getBytes().length);
        return response;
    }
}
```

Notes:
- The simplest approach I could think of to implement the REST controller
- Two constructors: one for app and another one for the test. This is because we're not using any CDI framework. 

And the repository:

```java
public class BookRepository {
    private static final Map<String, Book> BOOKS = new ConcurrentHashMap<>();

    public List<Book> getBooks() {
        return new ArrayList<>(BOOKS.values());
    }
}
```

Notes:
- Like I said earlier, we're not going to use any database, so let's keep the books in an in-memory collection. 
- Using `ConcurrentHashMap` implementation to not lock at `get` and `update` levels (by compared with sync hash map).

With the above changes, now our test will pass. 

# 3: Implement PUT method `/api/books/{title}`

The test:

```java
class AddBookByTitleIntegrationTest extends BaseIntegrationTest {

    @Test
    void testAddBookWhenEmptyRequestShouldReturn400() {
        RestAssured.given()
                .put("/api/books/Quijote")
                .then()
                .statusCode(400);
    }

    @Test
    void testAddBookShouldReturn201() {
        RestAssured.given().body("Cervantes")
                .put("/api/books/Quijote")
                .then()
                .statusCode(201);
    }
}
```

The controller:

```java
public class AddBookByTitleRestController implements RestController {

    private final BookService bookService;

    public AddBookByTitleRestController() {
        this(new BookService());
    }

    public AddBookByTitleRestController(BookService bookService) {
        this.bookService = bookService;
    }

    @Override
    public String path() {
        return "/api/books/(\\w)+$";
    }

    @Override
    public Method method() {
        return Method.PUT;
    }

    @Override
    public String handle(HttpExchange exchange) throws IOException {
        var parts = exchange.getRequestURI().getPath().split("/");
        String title = parts[parts.length - 1];
        String author = new String(exchange.getRequestBody().readAllBytes());
        if (author.isEmpty()) {
            exchange.sendResponseHeaders(400, 0);
            return null;
        }

        bookService.addBook(new Book(title, author));
        exchange.sendResponseHeaders(201, 0);
        return null;
    }
}
```

The "addBook" in the book service:

```java
public void addBook(Book book) {
    bookRepository.addBook(book);
}
```

And the one in the book repository:

```java
public void addBook(Book book) {
    BOOKS.put(book.title(), book);
}
```

With these changes, the test will now pass. 

# 4: Implement GET method `/api/books/{title}`

The test:

```java
class GetAuthorByBookIntegrationTest extends BaseIntegrationTest {

    @Test
    void testWhenBookDoesNotExistThenReturns404() {
        get("/api/books/Quijote")
                .then()
                .statusCode(404);
    }

    @Test
    void testWhenBookExistsThenReturnsAuthor() {
        givenExistingBook("Quijote", "Cervantes");
        get("/api/books/Quijote")
                .then()
                .statusCode(200)
                .body(is("Cervantes"));
    }
}
```

The controller:

```java
public class GetAuthorByBookRestController implements RestController {

    private final BookService bookService;

    public GetAuthorByBookRestController() {
        this(new BookService());
    }

    public GetAuthorByBookRestController(BookService bookService) {
        this.bookService = bookService;
    }

    @Override
    public String path() {
        return "/api/books/(\\w)+$";
    }

    @Override
    public Method method() {
        return Method.GET;
    }

    @Override
    public String handle(HttpExchange exchange) throws IOException {
        var parts = exchange.getRequestURI().getPath().split("/");
        String title = parts[parts.length - 1];
        String response = bookService.getAuthorByBook(title);
        if (response == null) {
            exchange.sendResponseHeaders(404, -1);
        } else {
            exchange.sendResponseHeaders(200, response.getBytes().length);
        }

        return response;
    }
}
```

And the new method "getAuthorByBook" in the service:

```java
public String getAuthorByBook(String title) {
    return getBooks().stream().filter(book -> book.title().equals(title))
            .map(Book::author)
            .findFirst()
            .orElse(null);
}
```

With these changes, the test will now pass. 

# 5: Implement DELETE method `/api/books/{title}`

The integration test:

```java
class DeleteBookByTitleIntegrationTest extends BaseIntegrationTest {

    @Test
    void testWhenBookDoesNotExistThenReturns404() {
        RestAssured.given()
                .delete("/api/books/Quijote")
                .then()
                .statusCode(404);
    }

    @Test
    void testWhenBookExistsThenReturn204() {
        RestAssured.given()
                .delete("/api/books/Quijote")
                .then()
                .statusCode(204);
    }
}
```

The controller:

```java
public class DeleteAuthorByBookRestController implements RestController {

    private final BookService bookService;

    public DeleteAuthorByBookRestController() {
        this(new BookService());
    }

    public DeleteAuthorByBookRestController(BookService bookService) {
        this.bookService = bookService;
    }

    @Override
    public String path() {
        return "/api/books/(\\w)+$";
    }

    @Override
    public Method method() {
        return Method.DELETE;
    }

    @Override
    public String handle(HttpExchange exchange) throws IOException {
        var parts = exchange.getRequestURI().getPath().split("/");
        String title = parts[parts.length - 1];
        boolean deleted = bookService.deleteBookByTitle(title);
        if (deleted) {
            exchange.sendResponseHeaders(204, -1);
        } else {
            exchange.sendResponseHeaders(404, -1);
        }

        return null;
    }
}
```

The new method "deleteBookByTitle" in the service:

```java
public boolean deleteBookByTitle(String title) {
    return bookRepository.deleteBookByTitle(title);
}
```

And the one in the repository:

```java
public boolean deleteBookByTitle(String title) {
    var book = BOOKS.remove(title);
    return book != null;
}
```

# 6: Implement POST method `/api/books`

The integration test:

```java
class AddBooksIntegrationTest extends BaseIntegrationTest {

    @Test
    void testAddBookWhenEmptyRequestShouldReturn400() {
        RestAssured.given()
                .post("/api/books/Quijote")
                .then()
                .statusCode(400);
    }

    @Test
    void testAddBookWhenWrongRequestShouldReturn400() {
        RestAssured.given().body("Quijote")
                .post("/api/books")
                .then()
                .statusCode(400);
    }

    @Test
    void testAddBookShouldReturn201() {
        RestAssured.given().body("Quijote-Cervantes")
                .post("/api/books")
                .then()
                .statusCode(201);
    }
}
```

The new controller:

```java
public class AddBooksRestController implements RestController {

    private final BookService bookService;

    public AddBooksRestController() {
        this(new BookService());
    }

    public AddBooksRestController(BookService bookService) {
        this.bookService = bookService;
    }

    @Override
    public String path() {
        return "/api/books$";
    }

    @Override
    public Method method() {
        return Method.POST;
    }

    @Override
    public String handle(HttpExchange exchange) throws IOException {
        String request = new String(exchange.getRequestBody().readAllBytes());
        String[] parts = request.split("-");
        if (parts.length != 2 || parts[0].isEmpty() || parts[1].isEmpty()) {
            exchange.sendResponseHeaders(400, 0);
            return null;
        }

        bookService.addBook(new Book(parts[0], parts[1]));
        exchange.sendResponseHeaders(201, 0);
        return null;
    }
}
```

With these changes, the test now will pass.

# Conclusion

There are a lot of points of improvement here, but I wanted to only give the simplest approach of a framework-less REST server using TDD and CLEAN patterns. 

The source code of the repository can be found [here](https://github.com/Sgitario/rest-server-java).
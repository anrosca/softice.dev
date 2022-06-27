---
title: "Bootiful error handling with @ControllerAdvices"
date: 2022-06-04T23:48:00+03:00
draft: false
author: "Andrei Rosca"
tags: ["Spring Boot"]
categories: ["Spring Boot"]
---

## Introduction

Today we're going to look at how to return pretty error responses for our REST APIs, using `Spring Boot's` controller advices. 
Even though controller advices are a well-known mechanism, no many projects use them to their full potential. In this article
we'll try to fix that.

### The REST api

Initially, we'll need a couple of HTTP endpoints, so that we can simulate some errors and see if we as users of that REST api can understand what went wrong. 
We'll use a simple HTTP api, exposing `CRUD` operations for movies.

Let's say we have the following controller:

```java
@RestController
@RequestMapping("/api/v1/movies")
public class MovieController {
    private final MovieService movieService;

    public MovieController(MovieService movieService) {
        this.movieService = movieService;
    }

    @GetMapping
    public List<MovieResponse> getAll() {
        return movieService.getAll();
    }

    @GetMapping("{id}")
    public MovieResponse getById(@PathVariable("id") Long id) {
        return movieService.getById(id);
    }

    @PostMapping
    public ResponseEntity<MovieResponse> create(@RequestBody @Validated CreateMovieRequest request) {
        MovieResponse createdMovie = movieService.create(request.toMovie());
        URI location =  MvcUriComponentsBuilder.fromMethodCall(MvcUriComponentsBuilder.on(getClass())
                        .getById(createdMovie.id()))
                .build()
                .toUri();
        return ResponseEntity.created(location)
                .body(createdMovie);
    }
}
```

As we can see, the controller does not have any sophisticated logic (and it should be like this ~~stop putting business logic in controllers~~), it just delegates to the service. To make things clearer,
let's take a look at the service as well.

```java
@Service
public class MovieService {
    private final MovieRepository movieRepository;

    public MovieService(MovieRepository movieRepository) {
        this.movieRepository = movieRepository;
    }

    @Transactional(readOnly = true)
    public List<MovieResponse> getAll() {
        return movieRepository.findAll()
                .stream()
                .map(MovieResponse::from)
                .toList();
    }

    @Transactional(readOnly = true)
    public MovieResponse getById(Long id) {
        Movie movie = movieRepository.findById(id)
                .orElseThrow(() -> new EntityNotFoundException("No movie with id:' " + id + "' was found."));
        return MovieResponse.from(movie);
    }

    @Transactional
    public MovieResponse create(Movie movieToCreate) {
        Movie createdMovie = movieRepository.save(movieToCreate);
        return MovieResponse.from(createdMovie);
    }
}
```

The service is also not that complicated, it has some basic operations like create and get movie by id and it just uses
a simple `Spring-Data-JPA` repository.

Using this simple REST api, let's try to simulate some errors to see how the default error responses will look like.
The simplest endpoint to call can be finding a movie by id. Let's see the successful scenario:

```http
GET http://localhost:8081/api/v1/movies/1
```

And we get the following response:

```json
HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Mon, 27 Jun 2022 20:00:48 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "id": 1,
  "name": "Pulp Fiction",
  "publishYear": 1994
}
```

Now let's try to simulate an error. If we look at our endpoint, we can see that the movie id has the type `java.lang.Long`. 
What will happen if the user will pass something that cannot be parsed as a `Long`? 

```java
@RestController
@RequestMapping("/api/v1/movies")
public class MovieController {
    //...
    @GetMapping("{id}")
    public MovieResponse getById(@PathVariable("id") Long id) {
        return movieService.getById(id);
    }
    //...
}
```

Let's find out. We'll use the string `'a'` as an invalid id.

```http
GET http://localhost:8081/api/v1/movies/a
```

And we get back the following response:

```json
HTTP/1.1 400 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Mon, 27 Jun 2022 20:08:57 GMT
Connection: close

{
  "timestamp": "2022-06-27T20:08:57.459+00:00",
  "status": 400,
  "error": "Bad Request",
  "path": "/api/v1/movies/a"
}
```

We got back an HTTP response with the status code `400 Bad request` with a json payload describing the error, 
all of that was done by `Spring Boot`, without any configuration.
Pretty good for a default, but the problem with that response is that it communicates almost nothing. Something's wrong
with our request, but what exactly? 

In our case, the request is quite simple and it's easy to figure out the problem, but
in cases where the endpoint which we're calling has a lot of `path` and `query` parameters, the root cause can be difficult to find.

If we look at the logs, we can spot an error:

```
2022-06-27 23:08:57.458  WARN 809940 --- [nio-8081-exec-5] .w.s.m.s.DefaultHandlerExceptionResolver : Resolved [org.springframework.web.method.annotation.MethodArgumentTypeMismatchException: Failed to convert value of type 'java.lang.String' to required type 'java.lang.Long'; nested exception is java.lang.NumberFormatException: For input string: "a"]
```

Basically a `MethodArgumentTypeMismatchException` was thrown, and in order to return a more meaningful response, conveying more information about what happened,
we need to handle this exception. `@RestControllerAdvice` to the rescue!

### The `GlobalExceptionHandler`

Since the `MethodArgumentTypeMismatchException` can be thrown from any controller, we'll add a default `@RestControllerAdvice` which will return an error response, describing in more
details what went wrong. We'll call it `GlobalExceptionHandler` and it will look like this:

```java
@Slf4j
@RestControllerAdvice
@Order(Ordered.LOWEST_PRECEDENCE)
public class GlobalExceptionHandler {
}
```

Notice the `@Order(Ordered.LOWEST_PRECEDENCE)` annotation. It is needed because we want our `GlobalExceptionHandler` to be the last one called, 
when a controller didn't handle an exception. Sometimes there are exceptions specific for a particular controller. In that case we want to allow the possibility of having
controller-specific controller-advices. Or in other words, we want the `GlobalExceptionHandler` to be the last controller-advice which is called, and allow controller-specific controller-advices
to be executed first.

Now, in order to return a pretty error response, we need a `DTO` class containing information about the error. We'll call it `ErrorResponse`, and it will look something like this:

```java
public record ErrorResponse(String statusCode, String path, List<String> messages) {
    @Builder
    public ErrorResponse {
    }
}
```

It has just 3 fields:
- `statusCode`: represents the error status code. Can be the HTTP status code or something similar to that.
- `path`: represents the request path of the called endpoint
- `messages`: an array of strings describing the error

## `MethodArgumentTypeMismatchException`

Let's handle our `MethodArgumentTypeMismatchException`. For that, we'll need to add an `@ExceptionHandler` in our `GlobalExceptionHandler`. It will look something like this:

```java
@Slf4j
@RestControllerAdvice
@Order(Ordered.LOWEST_PRECEDENCE)
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    public ResponseEntity<ErrorResponse> onMethodArgumentTypeMismatchException(MethodArgumentTypeMismatchException e, HttpServletRequest request) {
        MethodParameter parameter = e.getParameter();
        String message = "Parameter: '" + parameter.getParameterName() + "' is not valid. " + "Value '" + e.getValue()
                + "' could not be bound to type: '" + parameter.getParameterType()
                .getSimpleName()
                .toLowerCase()
                + "'";
        log.error("Exception while handling request: " + message, e);
        ErrorResponse errorResponse = ErrorResponse.builder()
                .statusCode(HttpStatus.BAD_REQUEST.name())
                .messages(List.of(message))
                .path(request.getServletPath())
                .build();
        return ResponseEntity.badRequest()
                .body(errorResponse);
    }
}
```

Basically we've extracted all the useful information (like the expected type and the invalid value) from the `MethodArgumentTypeMismatchException` and constructed an `ErrorResponse`,
which will be returned to the user.

Now, if we try to execute our invalid request again:

```http
GET http://localhost:8081/api/v1/movies/a
```

We should get the following response:

```json
HTTP/1.1 400 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Mon, 27 Jun 2022 20:16:30 GMT
Connection: close

{
  "statusCode": "BAD_REQUEST",
  "path": "/api/v1/movies/a",
  "messages": [
    "Parameter: 'id' is not valid. Value 'a' could not be bound to type: 'long'"
  ]
}
```

Now it's way more clear what's wrong. The value `a` is not a numeric value. There are some downsides though. One could argue that our error message is too revealing,
and that communicating to the user that we expect a `long` value can give a hint to a malicious user in what language our REST api was written in. 

We can argue that this is not a huge problem, we can easily change the error message to not include the expected type. Everything is up to us. 

## Missing query parameters

Let's modify our `getMovieById` endpoint and use a `query` parameter for the movie id instead of a `path` parameter.

```java
@RestController
@RequestMapping("/api/v1/movies")
public class MovieController {
    private final MovieService movieService;

    public MovieController(MovieService movieService) {
        this.movieService = movieService;
    }

    @GetMapping
    public MovieResponse getById(@RequestParam("id") Long id) {
        return movieService.getById(id);
    }
}
```

A successful requst will look like this:

```http
GET http://localhost:8081/api/v1/movies?id=1
```

We should get the following response:

```json
HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Mon, 27 Jun 2022 20:52:26 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "id": 1,
  "name": "Pulp Fiction",
  "publishYear": 1994
}
```

Looking good. What will happen if we'll pass an invalid value for the `id` query parameter? After all, the movie id is still a `long`. Let's find out:

```http
GET http://localhost:8081/api/v1/movies?id=a
```

We passed the value `'a'` as the movie id, which clearly there's no way it is a valid `long` value. We'll get back the following response:

```json
HTTP/1.1 400 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Mon, 27 Jun 2022 20:53:44 GMT
Connection: close

{
  "statusCode": "BAD_REQUEST",
  "path": "/api/v1/movies",
  "messages": [
    "Parameter: 'id' is not valid. Value 'a' could not be bound to type: 'long'"
  ]
}
```

Pretty nice, we got a `MethodArgumentTypeMismatchException` again, so our `GlobalExceptionHandler` did it's magic. Now the question is,
what will happen if we were to omit the `id` query parameter? Let's try it out:

```http
GET http://localhost:8081/api/v1/movies
```

We'll get the following HTTP response:

```json
HTTP/1.1 400 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Mon, 27 Jun 2022 20:54:32 GMT
Connection: close

{
  "timestamp": "2022-06-27T20:54:32.038+00:00",
  "status": 400,
  "error": "Bad Request",
  "path": "/api/v1/movies"
}
```

Not so great, we know our request is bad, but what exactly? If we look at the logs, we can see the following error:

```
2022-06-28 00:03:39.197  WARN 819720 --- [nio-8081-exec-1] .w.s.m.s.DefaultHandlerExceptionResolver : Resolved [org.springframework.web.bind.MissingServletRequestParameterException: Required request parameter 'id' for method parameter type Long is not present]
```

A `MissingServletRequestParameterException` was thrown, and we should handle it if we want a prettier error response. It will look something like this:

```java
@Slf4j
@RestControllerAdvice
@Order(Ordered.LOWEST_PRECEDENCE)
public class GlobalExceptionHandler {

    @ExceptionHandler(MissingServletRequestParameterException.class)
    public ResponseEntity<ErrorResponse> onMissingServletRequestParameterException(MissingServletRequestParameterException e,
                                                                                   HttpServletRequest request) {
        String message = "Parameter: '" + e.getParameterName() + "' of type " + e.getParameterType().toLowerCase(Locale.ROOT) + " is required but is missing";
        log.error("Exception while handling request: " + message, e);
        ErrorResponse errorResponse = ErrorResponse.builder()
                .statusCode(HttpStatus.BAD_REQUEST.name())
                .messages(List.of(message))
                .path(request.getServletPath())
                .build();
        return ResponseEntity.badRequest()
                .body(errorResponse);
    }
}
```

If we'll try to execute again our erroneous http request with the `id` query parameter missing, like shown below:

```http
GET http://localhost:8081/api/v1/movies
```

We should get the following error response:

```json
HTTP/1.1 400 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Mon, 27 Jun 2022 21:05:52 GMT
Connection: close

{
  "statusCode": "BAD_REQUEST",
  "path": "/api/v1/movies",
  "messages": [
    "Parameter: 'id' of type long is required but is missing"
  ]
}
```

Way better. From the response payload we can deduce what's wrong with our request.

## `MethodArgumentNotValidException`

Let's try to create a movie, to see what happens when its validation fails. Just to recap, the endpoint which allows users to create new movies looks like this:

```java
@RestController
@RequestMapping("/api/v1/movies")
public class MovieController {
    //...
    @PostMapping
    public ResponseEntity<MovieResponse> create(@RequestBody @Validated CreateMovieRequest request) {
        MovieResponse createdMovie = movieService.create(request.toMovie());
        URI location =  MvcUriComponentsBuilder.fromMethodCall(MvcUriComponentsBuilder.on(getClass())
                        .getById(createdMovie.id()))
                .build()
                .toUri();
        return ResponseEntity.created(location)
                .body(createdMovie);
    }
    //...
}
```

The request payload, is represented by a record called `CreateMovieRequest`, which can be seen below:

```java
public record CreateMovieRequest(@NotEmpty String name, @NotNull Integer publishYear) {
    public Movie toMovie() {
        return Movie.builder()
                .name(name)
                .publishYear(publishYear)
                .build();
    }
}
```

Basically in order to create a movie, the request payload has to have 2 fields, both of them are mandatory - `name` and `publishYear`. But what will happen if one of them is missing?
How will the error response look like? Let's try it out!

```http
POST http://localhost:8081/api/v1/movies
Content-Type: application/json

{
  "publishYear": 1994
}
```

And we get back the following response:

```json
HTTP/1.1 400 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Wed, 29 Jun 2022 19:58:40 GMT
Connection: close

{
  "timestamp": "2022-06-29T19:58:40.521+00:00",
  "status": 400,
  "error": "Bad Request",
  "path": "/api/v1/movies"
}
```

Not very descriptive. We've received a `HTTP 400` bad request, but we have no clue what's actually wrong with our request. Let's try to improve it a bit by adding the following property
in the `application.properties`:

```
server.error.include-binding-errors=always
```

Now, if we re-execute our invalid request:

```http
POST http://localhost:8081/api/v1/movies
Content-Type: application/json

{
  "publishYear": 1994
}
```

We'll get back the following response:

```json
HTTP/1.1 400 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Wed, 29 Jun 2022 20:08:07 GMT
Connection: close

{
  "timestamp": "2022-06-29T20:08:07.841+00:00",
  "status": 400,
  "error": "Bad Request",
  "errors": [
    {
      "codes": [
        "NotEmpty.createMovieRequest.name",
        "NotEmpty.name",
        "NotEmpty.java.lang.String",
        "NotEmpty"
      ],
      "arguments": [
        {
          "codes": [
            "createMovieRequest.name",
            "name"
          ],
          "arguments": null,
          "defaultMessage": "name",
          "code": "name"
        }
      ],
      "defaultMessage": "must not be empty",
      "objectName": "createMovieRequest",
      "field": "name",
      "rejectedValue": null,
      "bindingFailure": false,
      "code": "NotEmpty"
    }
  ],
  "path": "/api/v1/movies"
}
```

On one hand, this is definitely an improvement, since if we squint a bit, we can deduce that the validation of the field `name` from the request payload failed. It shouldn't be empty.
But still, this is an awfully bad error response. Imagine if the request contained not a single invalid field but a bunch of them. This error response will quickly start to look like a real mess,
so let's try to improve it.

```java
@Slf4j
@RestControllerAdvice
@Order(Ordered.LOWEST_PRECEDENCE)
public class GlobalExceptionHandler {
    private final MessageSource messageSource;

    public GlobalExceptionHandler(MessageSource messageSource) {
        this.messageSource = messageSource;
    }

    @ExceptionHandler({ MethodArgumentNotValidException.class })
    public ResponseEntity<ErrorResponse> onMethodArgumentNotValidException(MethodArgumentNotValidException e, HttpServletRequest request) {
        log.error("Exception while handling request.", e);
        BindingResult bindingResult = e.getBindingResult();
        List<String> errorMessages = new ArrayList<>();
        for (ObjectError error : bindingResult.getAllErrors()) {
            String resolvedMessage = messageSource.getMessage(error, Locale.US);
            if (error instanceof FieldError fieldError) {
                errorMessages.add(String.format("Field '%s' %s but value was '%s'", fieldError.getField(), resolvedMessage,
                        fieldError.getRejectedValue()));
            } else {
                errorMessages.add(resolvedMessage);
            }
        }
        ErrorResponse errorResponse = ErrorResponse.builder()
                .statusCode(HttpStatus.BAD_REQUEST.name())
                .messages(errorMessages)
                .path(request.getServletPath())
                .build();
        return ResponseEntity.badRequest()
                .body(errorResponse);
    }
    //...
}
```

We've added a new handler in our `GlobalExceptionHandler` for the `MethodArgumentNotValidException` exception, which will be thrown whenever the validation of the request payload will
fail. From it we can extract the binding errors, and format the in a more digestible or user-friendly way.

Now if we execute our incorrect request with the movie field name missing: 

```http
POST http://localhost:8081/api/v1/movies
Content-Type: application/json

{
  "publishYear": 1994
}
```

We'll get back the following response:

```json
HTTP/1.1 400 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Wed, 29 Jun 2022 20:13:21 GMT
Connection: close

{
  "statusCode": "BAD_REQUEST",
  "path": "/api/v1/movies",
  "messages": [
    "Field 'name' must not be empty but value was 'null'"
  ]
}
```

It's definitely more concise and clear, it explicitly says what field didn't pass the validation, without much clutter. 

## `ConstraintViolationException`

Let's get back to our get movie by id endpoint, which uses a query parameter. It is a good idea to validate it as well. For example, let's say that we expect our movie ids to be always
positive. In order to achieve that, we'll annotate our controller with the `@Validated` annotation, which will enable us to validate all method parameters.

To enforce the rule that movie ids are positive, we can just annotate our `id` query parameter with the `@Min(1)` annotation, like shown below:

```java
@RestController
@RequestMapping("/api/v1/movies")
@Validated
public class MovieController {
    private final MovieService movieService;

    public MovieController(MovieService movieService) {
        this.movieService = movieService;
    }

    @GetMapping
    public MovieResponse getById(@RequestParam("id") @Min(1) Long id) {
        return movieService.getById(id);
    }
    //...
}
```

Let's try to call the endpoint with an invalid id, and see what kind of error response we get back:

```http
GET http://localhost:8081/api/v1/movies?id=0
```

The error response will look like this:

```json
HTTP/1.1 500 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Wed, 29 Jun 2022 20:23:27 GMT
Connection: close

{
  "timestamp": "2022-06-29T20:23:27.593+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "path": "/api/v1/movies"
}
```

We've got an `HTTP 500` internal server error, and in the logs we can see the following exception:

```
javax.validation.ConstraintViolationException: getById.id: must be greater than or equal to 1
	at org.springframework.validation.beanvalidation.MethodValidationInterceptor.invoke(MethodValidationInterceptor.java:120) ~[spring-context-5.3.20.jar:5.3.20]
```

So in order to return a more meaningful response, we'll need to handle the `ConstraintViolationException` exception. Let's adjust our `GlobalExceptionHandler`, like shown in the 
code snippet below:

```java
@Slf4j
@RestControllerAdvice
@Order(Ordered.LOWEST_PRECEDENCE)
public class GlobalExceptionHandler {
    //...
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ErrorResponse> onConstraintViolationException(ConstraintViolationException e, HttpServletRequest request) {
        log.error("Exception while handling request.", e);
        Set<ConstraintViolation<?>> constraintViolations = e.getConstraintViolations();
        List<String> errorMessages = new ArrayList<>();
        for (ConstraintViolation<?> violation : constraintViolations) {
            errorMessages.add(String.format("Field '%s' %s but value was '%s'", getInvalidPropertyName(violation), violation.getMessage(),
                    violation.getInvalidValue()));
        }
        ErrorResponse errorResponse = ErrorResponse.builder()
                .statusCode(HttpStatus.BAD_REQUEST.name())
                .messages(errorMessages)
                .path(request.getServletPath())
                .build();
        return ResponseEntity.badRequest()
                .body(errorResponse);
    }

    private String getInvalidPropertyName(ConstraintViolation<?> violation) {
        return StreamSupport.stream(violation.getPropertyPath().spliterator(), false)
                .map(Path.Node::getName)
                .reduce((a, b) -> b)
                .orElse(violation.getPropertyPath().toString());
    }
    //...
}
```

This somewhat resembles how we handled the `MethodArgumentNotValidException`, but since it's a different exception, the handling has some differences. Let's see it in action. We'll call
again our endpoint, using an invalid movie id:

```http
GET http://localhost:8081/api/v1/movies?id=0
```

And the new error response should look like this:

```json
HTTP/1.1 400 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Wed, 29 Jun 2022 20:29:39 GMT
Connection: close

{
  "statusCode": "BAD_REQUEST",
  "path": "/api/v1/movies",
  "messages": [
    "Field 'id' must be greater than or equal to 1 but value was '0'"
  ]
}
```

Great success, the cause of the error is now obvious, the movie id query parameter has a value less than `1`.

## Not found exceptions

What will happen when the user will try to get an movie by its id, but the movie won't be found in the database? If we look at our `MovieService.getById()` method,
we can see that it will throw an `EntityNotFoundException` when the movie is not found in the database, like shown below:

```java
@Service
public class MovieService {
    //...
    @Transactional(readOnly = true)
    public MovieResponse getById(Long id) {
        Movie movie = movieRepository.findById(id)
                .orElseThrow(() -> new EntityNotFoundException("No movie with id:' " + id + "' was found."));
        return MovieResponse.from(movie);
    }
    //...
}
```

The `EntityNotFoundException`, is a custom exception, and its sole purpose is to signal that a specific entity was not found in the database. It can be seen below:

```java
public class EntityNotFoundException extends RuntimeException {
    public EntityNotFoundException(String message) {
        super(message);
    }
}
```

It's interesting to see how the error response will look like when this exception will be thrown. Let's try it out:

```http
GET http://localhost:8081/api/v1/movies?id=10
```

And we'll get back the following error response:

```json
HTTP/1.1 500 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Thu, 30 Jun 2022 05:52:43 GMT
Connection: close

{
  "timestamp": "2022-06-30T05:52:43.464+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "path": "/api/v1/movies"
}
```

As usual, the error response doesn't look pretty. Not only it doesn't describe properly what happened, but this time it doesn't blame the user but the server (since we got an `HTTP 500` internal server error).

The fix is simple and obvious, we'll need to handle the `EntityNotFoundException` in our `GlobalExceptionHandler`, something like this:

```java
@Slf4j
@RestControllerAdvice
@Order(Ordered.LOWEST_PRECEDENCE)
public class GlobalExceptionHandler {
    //...
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> oEntityNotFoundException(EntityNotFoundException e, HttpServletRequest request) {
        String message = e.getMessage();
        log.error("Exception while handling request: " + message, e);
        ErrorResponse errorResponse = ErrorResponse.builder()
                .statusCode(HttpStatus.NOT_FOUND.name())
                .messages(List.of(message))
                .path(request.getServletPath())
                .build();
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(errorResponse);
    }
    //...
}
```

This approach relies on the fact that whenever a entity won't be found in the database, an `EntityNotFoundException` will be thrown, containing a message stating which entity with which id
was not found. But it's easy to adopt this rule and make all controllers in our application follow it.

Now if we try to get a movie which doesn't exist in the database:

```http
GET http://localhost:8081/api/v1/movies?id=10
```

We will get the following response:

```json
HTTP/1.1 404 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Thu, 30 Jun 2022 05:58:19 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "statusCode": "NOT_FOUND",
  "path": "/api/v1/movies",
  "messages": [
    "No movie with id:' 10' was found."
  ]
}
```

Way better, the error is more concise this time.

## Controller-specific exceptions

The `GlobalExceptionHandler` we saw so far is intended to handle common exceptions, which can be thrown from all controllers. Sometimes though, it is needed to throw an exception, 
specific for a particular controller. In that case, a controller-specific controller-advice can be used.

For example, let's say that the `MovieController` can throw a `MovieAlreadyExistsException` when the user tries to create a movie, but it already exists in the database. If this 
exception is thrown only from the `MovieController`, then we'll create a controller-advice specific to this controller, like shown below:

```java
@RestControllerAdvice(assignableTypes = MovieController.class)
@Slf4j
@Order(Ordered.HIGHEST_PRECEDENCE)
public class MovieControllerAdvice {
    @ExceptionHandler(MovieAlreadyExistsException.class)
    public ResponseEntity<ErrorResponse> onMovieAlreadyExistsException(MovieAlreadyExistsException e, HttpServletRequest request) {
        String message = e.getMessage();
        log.error("Exception while handling request: " + message, e);
        ErrorResponse errorResponse = ErrorResponse.builder()
                .statusCode(HttpStatus.CONFLICT.name())
                .messages(List.of(message))
                .path(request.getServletPath())
                .build();
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(errorResponse);
    }
}
```

Notice the `@RestControllerAdvice(assignableTypes = MovieController.class)` annotation. It specifies that this controller-advice handles only exceptions from the `MovieController` (and it's subclasses),
but since we don't use the controller inheritance in our application, the `MovieController` is the only option. 

Also an important part is that we have to include the `@Order(Ordered.HIGHEST_PRECEDENCE)` annotation as well, to force the `MovieControllerAdvice` to be executed before the
`GlobalExceptionHandler` (because the `GlobalExceptionHandler` can have a generic, "catch-all" `@ExceptionHandler`).

## Putting it all together

The final version of our `GlobalExceptionHandler` will look something like this:

```java
@Slf4j
@RestControllerAdvice
@Order(Ordered.LOWEST_PRECEDENCE)
public class GlobalExceptionHandler {
    public static final String DEFAULT_ERROR_MESSAGE = "An unexpected exception occurred while processing the request";
    private final MessageSource messageSource;
    
    public GlobalExceptionHandler(MessageSource messageSource) {
        this.messageSource = messageSource;
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> onException(Exception e, HttpServletRequest request) {
        log.error("Exception while handling request", e);
        String errorMessage = e.getMessage() != null ? e.getMessage() : DEFAULT_ERROR_MESSAGE;
        ErrorResponse errorResponse = ErrorResponse.builder()
                .statusCode(HttpStatus.INTERNAL_SERVER_ERROR.name())
                .messages(List.of(errorMessage))
                .path(request.getServletPath())
                .build();
        return ResponseEntity.internalServerError()
                .body(errorResponse);
    }

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> oEntityNotFoundException(EntityNotFoundException e, HttpServletRequest request) {
        String message = e.getMessage();
        log.error("Exception while handling request: " + message, e);
        ErrorResponse errorResponse = ErrorResponse.builder()
                .statusCode(HttpStatus.NOT_FOUND.name())
                .messages(List.of(message))
                .path(request.getServletPath())
                .build();
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(errorResponse);
    }

    @ExceptionHandler(MissingServletRequestParameterException.class)
    public ResponseEntity<ErrorResponse> onMissingServletRequestParameterException(MissingServletRequestParameterException e,
                                                                                   HttpServletRequest request) {
        String message = "Parameter: '" + e.getParameterName() + "' of type " + e.getParameterType().toLowerCase(Locale.ROOT) + " is required but is missing";
        log.error("Exception while handling request: " + message, e);
        ErrorResponse errorResponse = ErrorResponse.builder()
                .statusCode(HttpStatus.BAD_REQUEST.name())
                .messages(List.of(message))
                .path(request.getServletPath())
                .build();
        return ResponseEntity.badRequest()
                .body(errorResponse);
    }

    @ExceptionHandler({MethodArgumentNotValidException.class })
    public ResponseEntity<ErrorResponse> onMethodArgumentNotValidException(MethodArgumentNotValidException e, HttpServletRequest request) {
        log.error("Exception while handling request.", e);
        BindingResult bindingResult = e.getBindingResult();
        List<String> errorMessages = new ArrayList<>();
        for (ObjectError error : bindingResult.getAllErrors()) {
            String resolvedMessage = messageSource.getMessage(error, Locale.US);
            if (error instanceof FieldError fieldError) {
                errorMessages.add(String.format("Field '%s' %s but value was '%s'", fieldError.getField(), resolvedMessage,
                        fieldError.getRejectedValue()));
            } else {
                errorMessages.add(resolvedMessage);
            }
        }
        ErrorResponse errorResponse = ErrorResponse.builder()
                .statusCode(HttpStatus.BAD_REQUEST.name())
                .messages(errorMessages)
                .path(request.getServletPath())
                .build();
        return ResponseEntity.badRequest()
                .body(errorResponse);
    }

    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ErrorResponse> onConstraintViolationException(ConstraintViolationException e, HttpServletRequest request) {
        log.error("Exception while handling request.", e);
        Set<ConstraintViolation<?>> constraintViolations = e.getConstraintViolations();
        List<String> errorMessages = new ArrayList<>();
        for (ConstraintViolation<?> violation : constraintViolations) {
            errorMessages.add(String.format("Field '%s' %s but value was '%s'", getInvalidPropertyName(violation), violation.getMessage(),
                    violation.getInvalidValue()));
        }
        ErrorResponse errorResponse = ErrorResponse.builder()
                .statusCode(HttpStatus.BAD_REQUEST.name())
                .messages(errorMessages)
                .path(request.getServletPath())
                .build();
        return ResponseEntity.badRequest()
                .body(errorResponse);
    }

    private String getInvalidPropertyName(ConstraintViolation<?> violation) {
        return StreamSupport.stream(violation.getPropertyPath().spliterator(), false)
                .map(Path.Node::getName)
                .reduce((a, b) -> b)
                .orElse(violation.getPropertyPath().toString());
    }

    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    public ResponseEntity<ErrorResponse> onMethodArgumentTypeMismatchException(MethodArgumentTypeMismatchException e, HttpServletRequest request) {
        MethodParameter parameter = e.getParameter();
        String message = "Parameter: '" + parameter.getParameterName() + "' is not valid. " + "Value '" + e.getValue()
                + "' could not be bound to type: '" + parameter.getParameterType()
                .getSimpleName()
                .toLowerCase()
                + "'";
        log.error("Exception while handling request: " + message, e);
        ErrorResponse errorResponse = ErrorResponse.builder()
                .statusCode(HttpStatus.BAD_REQUEST.name())
                .messages(List.of(message))
                .path(request.getServletPath())
                .build();
        return ResponseEntity.badRequest()
                .body(errorResponse);
    }
}
```

Notice that we've also added a generic, "catch-all" exception handler method (the one annotated with the `@ExceptionHandler(Exception.class)` annotation). It will be used for unexpected
exceptions, and it that case it'll return a `HTTP 500` error response and the error message will be the exception message of the caught exception. Probably that's not a good idea for
production environment and instead a more generic message should be used, but that exception should be definitely logged, so that it's easy to understand what went wrong.

## Conclusion

In this blog post we've explored some common exceptions which can be thrown from all controllers, like `MethodArgumentTypeMismatchException`, `MissingServletRequestParameterException`,
`MethodArgumentNotValidException` and `ConstraintViolationException`. We also saw that the default error responses `Spring Boot` generates are not that descriptive and helpful most of the time.
But this can be easily fixed using a powerful tool called `@RestControllerAdvice`s. By adding a couple of exception handlers in a controller-advice, we can obtain more readable and 
descriptive error responses which will help not only the consumers of our REST apis, but will help us - the developers of that api as well. No need to scroll through the mire of logs, if the
error response is self-explanatory.

The code examples can be found on [GitHub](https://github.com/anrosca/bootiful-error-handling)

---
title: "Introduction to declarative transaction management in Spring Framework"
date: 2022-05-10T08:54:47+03:00
draft: false
author: "Andrei Rosca"
tags: ["Spring Framework", "Declarative transaction management","Programmatic transaction management"]
categories: ["Spring Framework"]
---
## Introduction

In this blog post we are going to explore the internals of Spring's declarative transaction management. We'll start with the basics, and then we'll dive deeper, looking at the internals and some potential pitfalls which
we can run into. 

We'll be using:
- `Spring Boot 2.6.7`
- `Java 17`
- `Postgresql`
- `Spring Data JPA`

But first, let's discuss a bit why do we even bother with transactions in the first place?

## Why do we need transactions?

The most common reason for using transactions in an application is to maintain a high degree of data integrity and consistency.
If we're unconcerned about the quality of our data, we don't need to concern ourselves with transactions.

Transaction management is ubiquitous, it is present in every Java application which uses a database. The Spring Framework out of the box provides a lot of mechanisms to manage transactions and though 
it makes our lives easier, it is quite important to understand how it works and what happens under the hood since there are some pitfalls which can lead to undesired results.
Let's take a closer look at what transaction management mechanism Spring provides and how we can use them.

Transactions ensure data integrity trough ACID guarantees, which can be recapped as:

- Atomicity
    - Each transaction is "all or nothing". All SQL statements in a transaction (select, insert, update, merge or delete) is treated as a single unit. Either all statements are executed, or none of it is executed.
- Consistency
    - After a transaction, the database is guaranteed to be in a consistent state (all the integrity constraints will be satisfied)
- Isolation
    - Concurrent transactions don’t interfere with or affect one another. Well, almost. It is possible to have some interference, depending on the used transaction isolation level.
- Durability
    - Ensures that changes to your data made by successfully executed transactions will be saved, even in the event of system failure

Let's take a look at a practical example. Let's try to insert into the database the following `JPA` entity 3 times:

```java
@MappedSuperclass
public class AbstractEntity {
  @Id
  @GenericGenerator(name = "uuid-generator", strategy = "org.hibernate.id.UUIDGenerator")
  @GeneratedValue(generator = "uuid-generator")
  protected String id;

  protected AbstractEntity() {
  }

  public String getId() {
    return id;
  }

  public boolean equals(Object other) {
    if (!(other instanceof AbstractEntity otherEntity))
      return false;
    return Objects.equals(id, otherEntity.getId());
  }

  public int hashCode() {
    return getClass().hashCode();
  }
}

@Entity
@Table(name = "movies")
public class Movie extends AbstractEntity {
    private String name;

    protected Movie() {
    }

    private Movie(MovieBuilder builder) {
        this.name = builder.name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Movie{" +
                "name='" + name + '\'' +
                ", id='" + id + '\'' +
                '}';
    }

    public static MovieBuilder builder() {
        return new MovieBuilder();
    }

    public static class MovieBuilder {
        private String name;

        public MovieBuilder name(String name) {
            this.name = name;
            return this;
        }

        public Movie build() {
            return new Movie(this);
        }
    }
}
```

In order to do that, we can create the following `Spring Data JPA` repository:

```java
@Repository
public interface MovieRepository extends JpaRepository<Movie, String> {
}
```

We can use the `MovieRepository` presented above and try to insert 3 movies into the database, like shown below:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
    }

    @Bean
    public CommandLineRunner commandLineRunner(MovieService movieService) {
        return args -> {
            List<String> movieNames = List.of(
                    "Pulp fiction", "Joker", "Snatch"
            );
            List<Movie> savedMovies = movieService.saveMovies(movieNames);
            log.debug("Saved movies: {}", savedMovies);
        };
    }
}

@Service
class MovieService {
    private final MovieRepository movieRepository;

    public MovieService(MovieRepository movieRepository) {
        this.movieRepository = movieRepository;
    }

    public List<Movie> saveMovies(List<String> movieNames) {
        return movieNames.stream()
                .map(movieName -> Movie.builder()
                        .name(movieName)
                        .build()
                )
                .map(movieRepository::save)
                .toList();
    }
}
```

The question is, how many database transactions are executed by the `MovieService.saveMovies()` method?
The answer is `3`, because every call to `MovieRepository.save()` method creates a new transaction.

To make sure that that's really happening, we can set the `debug` log level for the `JpaTransactionManager` and in this
way we will get debug-level logs every time it opens, commits or rolls-back transactions.
Setting the debug log level can be accomplished by adding the following property in the `application.properties` file:

```properties
#Logging properties
logging.level.org.springframework.orm.jpa=debug
```

If we run the application now, we can observe the following in the logs:

```
2022-05-10 18:23:56.122 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Creating new transaction with name [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-10 18:23:56.122 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Opened new EntityManager [SessionImpl(1337659716<open>)] for JPA transaction
2022-05-10 18:23:56.125 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@2c34402]
2022-05-10 18:23:56.134 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Initiating transaction commit
2022-05-10 18:23:56.135 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Committing JPA transaction on EntityManager [SessionImpl(1337659716<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-10 18:23:56.144 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Closing JPA EntityManager [SessionImpl(1337659716<open>)] after transaction



2022-05-10 18:23:56.144 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Creating new transaction with name [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-10 18:23:56.145 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Opened new EntityManager [SessionImpl(1862946352<open>)] for JPA transaction
2022-05-10 18:23:56.145 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@50ff7063]
2022-05-10 18:23:56.145 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Initiating transaction commit
2022-05-10 18:23:56.145 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Committing JPA transaction on EntityManager [SessionImpl(1862946352<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-10 18:23:56.146 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Closing JPA EntityManager [SessionImpl(1862946352<open>)] after transaction


2022-05-10 18:23:56.146 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Creating new transaction with name [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-10 18:23:56.146 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Opened new EntityManager [SessionImpl(654299840<open>)] for JPA transaction
2022-05-10 18:23:56.147 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@d28c214]
2022-05-10 18:23:56.147 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Initiating transaction commit
2022-05-10 18:23:56.147 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Committing JPA transaction on EntityManager [SessionImpl(654299840<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-10 18:23:56.149 DEBUG 336867 --- [main] o.s.orm.jpa.JpaTransactionManager: Closing JPA EntityManager [SessionImpl(654299840<open>)] after transaction

```

We can observe that the sequence: `Creating new transaction with name [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT` and `Committing JPA transaction on EntityManager...` is present `3` times.
That means that every movie is saved in a separate transaction and this also means that the `MovieService.saveMovies()` method is
not atomic. If for example run into some issue inserting the third movie (for example a unique constraint is violated), only the third transaction will be rolled-back, and
we'll end up with 2 movies in the database. If we expected that the `MovieService.saveMovies()` is atomic, meaning it "inserts all movies or nothing", that's certainly not the case.

The log entry `Creating new transaction with name [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT` is quite interesting. 
It mentions that the transaction has the name `org.springframework.data.jpa.repository.support.SimpleJpaRepository.save` and that it also has some attributes like `PROPAGATION_REQUIRED` and `ISOLATION_DEFAULT`. We'll discuss these attributes later.

An interesting question is, how did we end up with `3` database transactions? Well, since we don't have any transaction management in our code, `Spring Data JPA` comes to the rescue. 
It notices that there aren't any active transactions for the current thread and it creates transactions for us. 

When we call the `MovieRepository.save()`, eventually that method will delegate to the `org.springframework.data.jpa.repository.support.SimpleJpaRepository.save()` method, which looks like this:

```java
package org.springframework.data.jpa.repository.support;

@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {
    //...
	@Transactional
	@Override
	public <S extends T> S save(S entity) {
		Assert.notNull(entity, "Entity must not be null.");
		if (entityInformation.isNew(entity)) {
			em.persist(entity);
			return entity;
		} else {
			return em.merge(entity);
		}
	}
    //...
}
```

As we can see, the `save()` method is annotated with the `@Transactional` annotation, indicating that the method should be executed with an active database transaction, and if there isn't an active transaction, it will create one. 

### How to fix it?

To make the `MovieService.saveMovies()` method atomic and obtain the "all or nothing" behavior, we can just annotate the method with `@Transactional` annotation. 

This time the `org.springframework.data.jpa.repository.support.SimpleJpaRepository.save()` won't create a new transaction every time it is called but it will notice that with the current thread an active transaction is associated and it will join it.
It will look like this:

```java
@Service
class MovieService {
    private final MovieRepository movieRepository;

    public MovieService(MovieRepository movieRepository) {
        this.movieRepository = movieRepository;
    }

    @Transactional
    public List<Movie> saveMovies(List<String> movieNames) {
        return movieNames.stream()
                .map(movieName -> Movie.builder()
                        .name(movieName)
                        .build()
                )
                .map(movieRepository::save)
                .toList();
    }
}
```

If we try to run the example this time, we should see the following in the logs:

```
2022-05-10 19:00:25.049 DEBUG 350995 --- [main] o.s.orm.jpa.JpaTransactionManager: Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovies]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-10 19:00:25.049 DEBUG 350995 --- [main] o.s.orm.jpa.JpaTransactionManager: Opened new EntityManager [SessionImpl(2139895366<open>)] for JPA transaction
2022-05-10 19:00:25.050 DEBUG 350995 --- [main] o.s.orm.jpa.JpaTransactionManager: Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@376b5cb2]
2022-05-10 19:00:25.056 DEBUG 350995 --- [main] o.s.orm.jpa.JpaTransactionManager: Found thread-bound EntityManager [SessionImpl(2139895366<open>)] for JPA transaction
2022-05-10 19:00:25.056 DEBUG 350995 --- [main] o.s.orm.jpa.JpaTransactionManager: Participating in existing transaction
2022-05-10 19:00:25.064 DEBUG 350995 --- [main] o.s.orm.jpa.JpaTransactionManager: Found thread-bound EntityManager [SessionImpl(2139895366<open>)] for JPA transaction
2022-05-10 19:00:25.064 DEBUG 350995 --- [main] o.s.orm.jpa.JpaTransactionManager: Participating in existing transaction
2022-05-10 19:00:25.065 DEBUG 350995 --- [main] o.s.orm.jpa.JpaTransactionManager: Found thread-bound EntityManager [SessionImpl(2139895366<open>)] for JPA transaction
2022-05-10 19:00:25.065 DEBUG 350995 --- [main] o.s.orm.jpa.JpaTransactionManager: Participating in existing transaction
2022-05-10 19:00:25.065 DEBUG 350995 --- [main] o.s.orm.jpa.JpaTransactionManager: Initiating transaction commit
2022-05-10 19:00:25.065 DEBUG 350995 --- [main] o.s.orm.jpa.JpaTransactionManager: Committing JPA transaction on EntityManager [SessionImpl(2139895366<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-10 19:00:25.079 DEBUG 350995 --- [main] o.s.orm.jpa.JpaTransactionManager: Closing JPA EntityManager [SessionImpl(2139895366<open>)] after transaction
```

Notice that the `Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovies]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT` sequence is present only once this time, meaning we have a single transaction, as we expected.

It's also interesting that the transaction name has changed. Not it's `inc.evil.spring.tx.management.MovieService.saveMovies` which is precisely the fully qualified method name which was annotated with the `@Transactional` annotation.

Also in the logs we can see `Participating in existing transaction` `3` times, meaning that the `org.springframework.data.jpa.repository.support.SimpleJpaRepository.save()` method has joined the transaction created by the `inc.evil.spring.tx.management.MovieService.saveMovies()` 
method `3` times.

Since `Spring Data Jpa` as a last resort creates transactions for us, during the following examples we are going to use `Hibernate`'s `EntityManager` directly because it doesn't create any transactions and at the same time some of its methods throw exceptions if they're called
without an active transaction.

## Types of transaction management

The Spring Framework provides us with 2 types of transaction management:
- Programmatic transaction management
    - With this approach we manually create, commit and roll-back transactions
- Declarative transaction management
    - With this approach, we just declare that we want a transaction, via an annotation and Spring will take care of the rest 

Let's take a look at how both of these approaches look like in practice.

### Programmatic transaction management

As we mentioned, by using the programmatic transaction management approach, we will have to manually create and commit transactions. This is not the preferred approach nowadays, but it does have its use-cases.
Before showing any code examples, it's worth mentioning that when it comes to transaction management (doesn't matter declarative or programmatic), the entry-point is a spring bean of type `PlatformTransactionManager`
with the name `transactionManager`.

There are different implementations of the `PlatformTransactionManager` interface, depending on the environment we're using. Some noteworthy implementations are:
- `org.springframework.orm.jpa.JpaTransactionManager`: if we intend to use `Hibernate` (with or without `Spring Data Jpa`)
- `org.springframework.jdbc.datasource.DataSourceTransactionManager`: if we intend to use `sping-jdbc` (with the `JdbcTemplate`) or `Springh Data Jdbc`
- `org.springframework.transaction.jta.JtaTransactionManager`: if we intend to use distributed (or `XA`) transactions (meaning transactions spanning more than one transactional resource, like a database and a message queue). Although `XA` transactions look nice, it's considered a relic
of the past and it's best to be avoided since it has some problems and is not supported by many modern technologies (like `Apache Kafka` for example).

Using the programmatic transaction management involves using directly the `PlatformTransactionManager`. As we can see below, it's rather cumbersome to use.
The first thing we need to do is to instantiate a `DefaultTransactionDefinition` which we can use to specify the desired propagation behavior, transaction name, the isolation level, the transaction timeout and the transaction read-only flag.
After that by calling the `PlatformTransactionManager.getTransaction(TransactionDefinition)` method we effectively start the transaction with the desired attributes, execute the unit of work, and then we can try to commit it. 
If at some point the transaction was marked as rollback-only, it cannot be committed and the only possible outcome is a transaction rollback.

```java
@Service
class ProgrammaticTxMovieService {
    private final EntityManager entityManager;
    private final PlatformTransactionManager transactionManager;

    public ProgrammaticTxMovieService(EntityManager entityManager, PlatformTransactionManager transactionManager) {
        this.entityManager = entityManager;
        this.transactionManager = transactionManager;
    }

    public Movie saveMovie(String movieName) {
        DefaultTransactionDefinition definition = new DefaultTransactionDefinition();
        definition.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        definition.setName("saveMovie");
        TransactionStatus transaction = transactionManager.getTransaction(definition);
        try {
            Movie movie = Movie.builder()
                    .name(movieName)
                    .build();
            entityManager.persist(movie);
            return movie;
        } catch (Exception e) {
            transaction.setRollbackOnly();
            throw e;
        } finally {
            transactionManager.commit(transaction);
        }
    }
}
```

It's worth mentioning that injecting the `EntityManager` directly into a service is kind of an unorthodox approach since services shouldn't directly talk to the database but should delegate to a `DAO` instead. The rule was broken for the sake of brevity.

A slightly simpler approach is to use the `TransactionTemplate` class which basically abstracts away the commit and rollback logic. Instead or writing this logic ourselves, we pass a callback. If the callback will not throw any exceptions, the transaction will be committed,
otherwise a rollback will happen.

```java
  @Service
  class TransactionTemplateProgrammaticTxMovieService {
    private final EntityManager entityManager;
    private final PlatformTransactionManager transactionManager;
  
    public TransactionTemplateProgrammaticTxMovieService(EntityManager entityManager, PlatformTransactionManager transactionManager) {
      this.entityManager = entityManager;
      this.transactionManager = transactionManager;
    }
  
    public Movie saveMovie(String movieName) {
      DefaultTransactionDefinition definition = new DefaultTransactionDefinition();
      definition.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
      definition.setName("saveMovie");
      TransactionTemplate transactionTemplate = new TransactionTemplate(transactionManager, definition);
      return transactionTemplate.execute(status -> {
        Movie movie = Movie.builder()
                .name(movieName)
                .build();
        entityManager.persist(movie);
        return movie;
      });
    }
  }
```

### Declarative transaction management

With declarative transaction management, we just specify for what methods we want transactional behavior by annotating them with the `@Transactional` annotation and Spring will take care of the rest. It works something like this:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      List<String> movieNames = List.of(
              "Pulp fiction", "Joker", "Snatch"
      );
      List<Movie> savedMovies = movieService.saveMovies(movieNames);
      log.debug("Saved movies: {}", savedMovies);
    };
  }
}

@Service
class MovieService {
    private final MovieRepository movieRepository;

    public MovieService(MovieRepository movieRepository) {
        this.movieRepository = movieRepository;
    }

    @Transactional
    public List<Movie> saveMovies(List<String> movieNames) {
        return movieNames.stream()
                .map(movieName -> Movie.builder()
                        .name(movieName)
                        .build()
                )
                .map(movieRepository::save)
                .toList();
    }
}
```

But how exactly does it work? We just annotated a method with an annotation and somehow that method now executes within a transaction. But who starts the transaction? Who commits or rolls-back it?

The answer is simple, it's a proxy! 

Just as a refresher, the proxy pattern looks like this:

{{< figure src="proxy_2.png" alt="The proxy pattern" >}}

We have the `Subject` interface with the implementation `Real Subject`.

We also have a `Proxy` which implements the `Subject` interface, so the proxy looks like a real `Subject` but it adds additional behavior like caching, logging, security or even transactions.
The proxy also has a reference to the real implementation. 

The client, though unaware of this, will use the `Proxy` thinking it's "the real thing".

Let's try to log the actual class name of the `MovieService`, and see what's really going on.

```java
    @Bean
    public CommandLineRunner commandLineRunner(MovieService movieService) {
        return args -> {
            log.debug("MovieService actual class name: {}", movieService.getClass().getName());
        };
    }
```

then, in the logs we will see something like this:

```
2022-05-11 09:01:10.367 DEBUG 498664 --- [main] SpringDeclarativeTxManagementApplication: MovieService actual class name: inc.evil.spring.tx.management.MovieService$$EnhancerBySpringCGLIB$$38b14d5
```

We can observe that the actual class name is `inc.evil.spring.tx.management.MovieService$$EnhancerBySpringCGLIB$$38b14d5`, but in our codebase we have a simple `MovieService`.

Because our `MovieService` has methods annotated with `@Transactional` annotation, Spring has created a CGLib proxy for our service and all the logic regarding transaction creation, commit or roll-back is executed precisely by that proxy.

So, whenever we have a service which has methods annotated with the `@Transactional` annotation, Spring will create a proxy for that service and put it in the `ApplicationContext`. Wherever we want to inject that service, a proxy will be injected instead.
The proxy will intercept all the `public` method calls, check if the method needs a transaction and if so, it will create it and then it will delegate to the original `MovieService`. After the original method from the `MovieService` was executed, 
the proxy will then commit or rollback the transaction, depending on what (if any) exceptions were thrown.

As a sequence diagram, the flow looks like this:

{{< figure src="proxy_sequence.png" alt="The proxy pattern" >}}

We can summarize the flow like this:
- The `CommandLineRunner` calls `MovieService.saveMovies()` method. What the `CommandLineRunner` doesn't know is that it actually has a reference to a proxy. 
So in reality the `CommandLineRunner` is invoking a proxy (a class generated by CGLib in runtime, which looks like a `MovieService` but it's a completely different class called `MovieService$$EnhancerBySpringCGLIB$$38b14d5`)
- The proxy (the `MovieService$$EnhancerBySpringCGLIB$$38b14d5` class) sees that the invoked method (the `saveMovies` method) is annotated with the `@Transactional` annotation, so we need a transaction most likely.
It then checks if with the current thread there is a transaction associated already. If the current thread doesn't have an associated transaction, the proxy will create one. 
To be honest, the behavior of the proxy in this case depends on the configured transaction propagation level, which we'll discuss later. But by default, if there isn't a transaction, the proxy will create one.
- After the proxy has created the transaction, it will call the original method (the `MovieService.saveMovies()`, or target method). At this point we have an active transaction so the original `MovieService.saveMovies()`
method executes transactionally.
- After the target `MovieService.saveMovies()` method has finished its execution, the flow goes back to the proxy. The proxy checks if the criteria for a commit are met and if so, it commits the transaction. Otherwise we'll
have a transaction rollback. We'll discuss later in what cases the proxy decides to commit the transaction and when it rolls-back.
- After that the flow goes back to the original caller, the `CommandLineRunner` in our case

We can also annotate the whole class with the `@Transactional` and in this cass, all public method will be transactional, like shown below:

```java
@Service
@Transactional
class MovieService {
    private final MovieRepository movieRepository;

    public MovieService(MovieRepository movieRepository) {
        this.movieRepository = movieRepository;
    }

    public List<Movie> saveMovies(List<String> movieNames) {
        return movieNames.stream()
                .map(movieName -> Movie.builder()
                        .name(movieName)
                        .build()
                )
                .map(movieRepository::save)
                .toList();
    }
}
```

### `@Transactional` method visibility

By default, only `public` methods can be annotated with the `@Transactional` annotation. What will happen if we try to make a `@Transactional` method `package-private` or `protected`? Let's have a look (notice that the `MovieService.saveMovie` method is `package-private`):

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
  private final EntityManager entityManager;

  public MovieService(EntityManager entityManager) {
    this.entityManager = entityManager;
  }

  @Transactional
  Movie saveMovie(String movieName) {
    Movie movie = Movie.builder()
            .name(movieName)
            .build();
    entityManager.persist(movie);
    return movie;
  }
}
```

We'll get a `TransactionRequiredException` thrown by the `EntityManager.persist()` method because we don't have an active transaction (see logs below):

```
2022-05-12 12:51:40.418  INFO 1079474 --- [main] SpringDeclarativeTxManagementApplication : Started SpringDeclarativeTxManagementApplication in 1.079 seconds (JVM running for 1.355)
2022-05-12 12:51:40.420  INFO 1079474 --- [main] ConditionEvaluationReportLoggingListener : 

Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2022-05-12 12:51:40.428 ERROR 1079474 --- [main] o.s.boot.SpringApplication               : Application run failed

java.lang.IllegalStateException: Failed to execute CommandLineRunner
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:780) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.callRunners(SpringApplication.java:761) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:310) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1312) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1301) ~[spring-boot-2.6.7.jar:2.6.7]
	at inc.evil.spring.tx.management.SpringDeclarativeTxManagementApplication.main(SpringDeclarativeTxManagementApplication.java:35) ~[classes/:na]
Caused by: javax.persistence.TransactionRequiredException: No EntityManager with actual transaction available for current thread - cannot reliably process 'persist' call
	at org.springframework.orm.jpa.SharedEntityManagerCreator$SharedEntityManagerInvocationHandler.invoke(SharedEntityManagerCreator.java:295) ~[spring-orm-5.3.19.jar:5.3.19]
	at jdk.proxy2/jdk.proxy2.$Proxy80.persist(Unknown Source) ~[na:na]
	at inc.evil.spring.tx.management.MovieService.saveMovie(SpringDeclarativeTxManagementApplication.java:162) ~[classes/:na]
	at inc.evil.spring.tx.management.SpringDeclarativeTxManagementApplication.lambda$commandLineRunner$0(SpringDeclarativeTxManagementApplication.java:41) ~[classes/:na]
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:777) ~[spring-boot-2.6.7.jar:2.6.7]
	... 5 common frames omitted
```

It turns out that we can enable the transactional behavior for `package-private` and `protected` methods (`private` is the exception) by adding a bit of configuration. 

To do this, we should add in our Java configuration the `@EnableTransactionManagement(proxyTargetClass = true)` and add in the application context the following spring bean:

```java
    @Bean
    public TransactionAttributeSource transactionAttributeSource() {
        return new AnnotationTransactionAttributeSource(false);
    }
```

The `TransactionAttributeSource` constructor has a parameter called `publicMethodsOnly` (and by default it is set to `true`) which specifies if only `public` methods should have the transactional behavior. By setting this flag to `false`, we obtain transactional behavior for `package-private` and
`protected` methods as well.

Also since `Spring-Boot` registers by default this bean, we'll have to add in the `application.properties` file the following property, in order to allow bean-definition overriding:
```properties
spring.main.allow-bean-definition-overriding=true
```
The complete example looks like this:

```java
@Slf4j
@SpringBootApplication
@EnableTransactionManagement(proxyTargetClass = true)
public class SpringDeclarativeTxManagementApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
    }

    @Bean
    public CommandLineRunner commandLineRunner(MovieService movieService) {
        return args -> {
            movieService.saveMovie("Pulp fiction");
        };
    }

    @Bean
    public TransactionAttributeSource transactionAttributeSource() {
        return new AnnotationTransactionAttributeSource(false);
    }
}

@Service
@Slf4j
class MovieService {
  private final EntityManager entityManager;

  public MovieService(EntityManager entityManager) {
    this.entityManager = entityManager;
  }

  @Transactional
  Movie saveMovie(String movieName) {
    Movie movie = Movie.builder()
            .name(movieName)
            .build();
    entityManager.persist(movie);
    return movie;
  }
}
```

If we check the logs, we can observe that it indeed works:

```
2022-05-12 13:02:15.492 DEBUG 1082568 --- [main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-12 13:02:15.493 DEBUG 1082568 --- [main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(787497403<open>)] for JPA transaction
2022-05-12 13:02:15.494 DEBUG 1082568 --- [main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@41e8d917]
2022-05-12 13:02:15.500 DEBUG 1082568 --- [main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction commit
2022-05-12 13:02:15.501 DEBUG 1082568 --- [main] o.s.orm.jpa.JpaTransactionManager        : Committing JPA transaction on EntityManager [SessionImpl(787497403<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)

```

It is also possible to make the `@Transactional` annotation work for `private` methods (using weaving or bytecode-instrumentation), we'll describe this in another blog post.

### Commit and roll-back rules

By default (though this can be configured) the transaction will be committed if:
- No exceptions were thrown
- Checked exceptions were thrown

In other cases (if unchecked exceptions were thrown, meaning instances or subclasses of `RuntimeException`), the transaction will be rolled-back.

It is surprising that the transaction is committed in case of checked exceptions. The rationale is that checked exceptions are considered business exceptions and we should check the business rules to see if we do need a rollback or not.

Let's try to test it! What do you think will happen when we'll call the `MovieService.saveMovies()` method shown below?

```java
@Service
class MovieService {
    @Transactional
    public List<Movie> saveMovies(List<String> movieNames) {
        throw new IllegalArgumentException();
    }
}

```

Since `IllegalArgumentException` extends `RuntimeException`, it is considered an unchecked exception and the transaction will be rolled-back. Let's look at the logs:

```
2022-05-11 09:22:17.287 DEBUG 505977 --- [main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovies]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-11 09:22:17.287 DEBUG 505977 --- [main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(1159911315<open>)] for JPA transaction
2022-05-11 09:22:17.288 DEBUG 505977 --- [main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@37fef327]
2022-05-11 09:22:17.292 DEBUG 505977 --- [main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction rollback
2022-05-11 09:22:17.292 DEBUG 505977 --- [main] o.s.orm.jpa.JpaTransactionManager        : Rolling back JPA transaction on EntityManager [SessionImpl(1159911315<open>)]
2022-05-11 09:22:17.293 DEBUG 505977 --- [main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(1159911315<open>)] after transaction
2022-05-11 09:22:17.294  INFO 505977 --- [main] ConditionEvaluationReportLoggingListener : 

Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2022-05-11 09:22:17.303 ERROR 505977 --- [main] o.s.boot.SpringApplication               : Application run failed

java.lang.IllegalStateException: Failed to execute CommandLineRunner
```

The transaction was indeed rolled-back.

Now let's see what happens if we will use a checked-exception:

```java
@Service
class MovieService {
    @Transactional
    public List<Movie> saveMovies(List<String> movieNames) throws IOException {
        throw new IOException();
    }
}
```

This time we throw a `IOException` which extends from `Exception` which makes it a checked-exception. In this case, even though an exception was thrown out of the `@Transactional` method, we will have a commit, as seen in the logs below:

```
2022-05-11 09:26:23.561 DEBUG 507376 --- [main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovies]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-11 09:26:23.562 DEBUG 507376 --- [main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(347691330<open>)] for JPA transaction
2022-05-11 09:26:23.563 DEBUG 507376 --- [main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@775f15fd]
2022-05-11 09:26:23.566 DEBUG 507376 --- [main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction commit
2022-05-11 09:26:23.566 DEBUG 507376 --- [main] o.s.orm.jpa.JpaTransactionManager        : Committing JPA transaction on EntityManager [SessionImpl(347691330<open>)]
2022-05-11 09:26:23.567 DEBUG 507376 --- [main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(347691330<open>)] after transaction
2022-05-11 09:26:23.567  INFO 507376 --- [main] ConditionEvaluationReportLoggingListener : 

Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2022-05-11 09:26:23.575 ERROR 507376 --- [main] o.s.boot.SpringApplication               : Application run failed

java.lang.IllegalStateException: Failed to execute CommandLineRunner
```

### How to configure the commit-rollback behavior?

It is possible to specify the desired transaction outcome by using the following attributes of the `@Transactional annotation`:
- `@Transactional(rollbackFor = ...)`: an array of exception classes for which we want the transaction rolled-back. If we use for example `@Transactional(rollbackFor = Exception.class)`, then we'll have a rollback if the `@Transactional` method throws an `Exception` or any of its subclasses
- `@Transactional(rollbackForClassName = "...")`: same as above but this time the exception class name is specified as a string. Wildcards are not supported, but we can omit the package and use only the class name, 
      like this: `@Transactional(noRollbackForClassName = "IOException")` or like this `@Transactional(noRollbackForClassName = "java.io.IOException")` it's the same thing
- `@Transactional(noRollbackFor = ...)`: an array of exception classes for which we want the transaction committed.
- `@Transactional(noRollbackForClassName = "...")`: same as above but this time the exception class name is specified as a string.

Let's try to test it. What do you think will happen when we'll call the `MovieService.saveMovies()` method shown below?

```java
@Service
class MovieService {
    @Transactional(rollbackFor = IOException.class)
    public List<Movie> saveMovies(List<String> movieNames) throws IOException {
        throw new IOException();
    }
}
```

The transaction will be rolled-back. Even though a checked-exception was thrown, we specified that we want a rollback in case of an `IOException` is thrown, and that's exactly what we've got (see the logs below):

```
2022-05-11 10:02:49.324 DEBUG 519567 --- [main] o.s.orm.jpa.JpaTransactionManager: Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovies]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,-java.io.IOException
2022-05-11 10:02:49.324 DEBUG 519567 --- [main] o.s.orm.jpa.JpaTransactionManager: Opened new EntityManager [SessionImpl(1207093026<open>)] for JPA transaction
2022-05-11 10:02:49.326 DEBUG 519567 --- [main] o.s.orm.jpa.JpaTransactionManager: Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@7608a838]
2022-05-11 10:02:49.330 DEBUG 519567 --- [main] o.s.orm.jpa.JpaTransactionManager: Initiating transaction rollback
2022-05-11 10:02:49.330 DEBUG 519567 --- [main] o.s.orm.jpa.JpaTransactionManager: Rolling back JPA transaction on EntityManager [SessionImpl(1207093026<open>)]
2022-05-11 10:02:49.331 DEBUG 519567 --- [main] o.s.orm.jpa.JpaTransactionManager: Closing JPA EntityManager [SessionImpl(1207093026<open>)] after transaction
```

Now let's do the opposite with an unchecked-exception. What do you think will happen when we'll call the method shown below?

```java
@Service
class MovieService {
    @Transactional(noRollbackFor = IllegalArgumentException.class)
    public List<Movie> saveMovies(List<String> movieNames) {
        throw new IllegalArgumentException();
    }
}
```

The transaction will be committed since we've specified that we don't want to rollback in case an `IllegalArgumentException` exception is thrown (see the logs below):

```
2022-05-11 10:06:57.206 DEBUG 520985 --- [main] o.s.orm.jpa.JpaTransactionManager: Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovies]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,+java.lang.IllegalArgumentException
2022-05-11 10:06:57.206 DEBUG 520985 --- [main] o.s.orm.jpa.JpaTransactionManager: Opened new EntityManager [SessionImpl(1692381981<open>)] for JPA transaction
2022-05-11 10:06:57.207 DEBUG 520985 --- [main] o.s.orm.jpa.JpaTransactionManager: Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@2cd3fc29]
2022-05-11 10:06:57.210 DEBUG 520985 --- [main] o.s.orm.jpa.JpaTransactionManager: Initiating transaction commit
2022-05-11 10:06:57.211 DEBUG 520985 --- [main] o.s.orm.jpa.JpaTransactionManager: Committing JPA transaction on EntityManager [SessionImpl(1692381981<open>)]
2022-05-11 10:06:57.211 DEBUG 520985 --- [main] o.s.orm.jpa.JpaTransactionManager: Closing JPA EntityManager [SessionImpl(1692381981<open>)] after transaction
```

### Is programmatic transaction management equivalent to the declarative one?

What if we take a method which uses the declarative transaction management approach and try to rewrite it to use the programmatic approach? Do we get the same behavior? Let's have a look:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final EntityManager entityManager;

    public MovieService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public Movie saveMovie(String movieName) throws IOException {
        Movie movie = Movie.builder()
                .name(movieName)
                .build();
        entityManager.persist(movie);
        offendingMethod();
        return movie;
    }

  private void offendingMethod() throws IOException {
    throw new IOException();
  }
}
```

The method throws an `IOException`, which is a checked-exception so that means that the transaction will be committed, even though the method thew an exception (see the logs below):

```
2022-05-12 10:26:06.788 DEBUG 1024681 --- [main] o.s.orm.jpa.JpaTransactionManager: Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-12 10:26:06.788 DEBUG 1024681 --- [main] o.s.orm.jpa.JpaTransactionManager: Opened new EntityManager [SessionImpl(1871084300<open>)] for JPA transaction
2022-05-12 10:26:06.790 DEBUG 1024681 --- [main] o.s.orm.jpa.JpaTransactionManager: Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@be6d228]
2022-05-12 10:26:06.797 DEBUG 1024681 --- [main] o.s.orm.jpa.JpaTransactionManager: Initiating transaction commit
2022-05-12 10:26:06.798 DEBUG 1024681 --- [main] o.s.orm.jpa.JpaTransactionManager: Committing JPA transaction on EntityManager [SessionImpl(1871084300<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-12 10:26:06.810 DEBUG 1024681 --- [main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(1871084300<open>)] after transaction
2022-05-12 10:26:06.811  INFO 1024681 --- [main] ConditionEvaluationReportLoggingListener : 

Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2022-05-12 10:26:06.819 ERROR 1024681 --- [main] o.s.boot.SpringApplication               : Application run failed

java.lang.IllegalStateException: Failed to execute CommandLineRunner
```

Now let's rewrite the method to use the programmatic transaction management approach using the `TransactionTemplate`, as shown below:

```java
@Service
@Slf4j
class MovieService {
  private final EntityManager entityManager;
  private final PlatformTransactionManager transactionManager;

  public MovieService(EntityManager entityManager, PlatformTransactionManager transactionManager) {
    this.entityManager = entityManager;
    this.transactionManager = transactionManager;
  }

  public Movie saveMovie(String movieName) throws IOException {
    DefaultTransactionDefinition definition = new DefaultTransactionDefinition();
    definition.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
    definition.setName("saveMovie");
    TransactionTemplate transactionTemplate = new TransactionTemplate(transactionManager, definition);
    return transactionTemplate.execute(status -> {
      Movie movie = Movie.builder()
              .name(movieName)
              .build();
      entityManager.persist(movie);
      offendingMethod(); //does not compile
      return movie;
    });
  }

  private void offendingMethod() throws IOException {
    throw new IOException();
  }
}
```

Well, the code above does not compile since the `TransactionTemplate.execute(TransactionCallback callback)` method have a parameter of type `TransactionCallback`, which is a functional interface which doesn't declare any checked exceptions and thus, our lambda implementation can't
throw any checked exceptions. The `TransactionCallback` interface can be seen below

```java
package org.springframework.transaction.support;

@FunctionalInterface
public interface TransactionCallback<T> {
    @Nullable
    T doInTransaction(TransactionStatus status);
}

```

What if we rewrite the method, replacing the lambda expression with an anonymous inner class and annotate it with a Lombok's `@SneakyThrows` annotation:

```java
@Service
@Slf4j
class MovieService {
    private final EntityManager entityManager;
    private final PlatformTransactionManager transactionManager;

    public MovieService(EntityManager entityManager, PlatformTransactionManager transactionManager) {
        this.entityManager = entityManager;
        this.transactionManager = transactionManager;
    }

    public Movie saveMovie(String movieName) throws IOException {
        DefaultTransactionDefinition definition = new DefaultTransactionDefinition();
        definition.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        definition.setName("saveMovie");
        TransactionTemplate transactionTemplate = new TransactionTemplate(transactionManager, definition);
        return transactionTemplate.execute(new TransactionCallback<Movie>() {
            @Override
            @SneakyThrows
            public Movie doInTransaction(TransactionStatus status) {
                Movie movie = Movie.builder()
                        .name(movieName)
                        .build();
                entityManager.persist(movie);
                offendingMethod();
              return movie;
            }
        });
    }

  private void offendingMethod() throws IOException {
    throw new IOException();
  }
}
```

The code compiles fine but if we try to run it, in this case the transaction will be rolled-back (`Rolling back JPA transaction on EntityManager [SessionImpl(357751318<open>)]`) because the `TransactionTemplate` doesn't expect any checked-exceptions to be thrown:

```
2022-05-12 10:44:22.556 DEBUG 1030147 --- [main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-12 10:44:22.556 DEBUG 1030147 --- [main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(357751318<open>)] for JPA transaction
2022-05-12 10:44:22.557 DEBUG 1030147 --- [main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@54be6213]
2022-05-12 10:44:22.562 DEBUG 1030147 --- [main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction rollback
2022-05-12 10:44:22.563 DEBUG 1030147 --- [main] o.s.orm.jpa.JpaTransactionManager        : Rolling back JPA transaction on EntityManager [SessionImpl(357751318<open>)]
2022-05-12 10:44:22.563 DEBUG 1030147 --- [main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(357751318<open>)] after transaction
2022-05-12 10:44:22.564  INFO 1030147 --- [main] ConditionEvaluationReportLoggingListener : 

Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2022-05-12 10:44:22.572 ERROR 1030147 --- [main] o.s.boot.SpringApplication               : Application run failed

java.lang.IllegalStateException: Failed to execute CommandLineRunner
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:780) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.callRunners(SpringApplication.java:761) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:310) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1312) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1301) ~[spring-boot-2.6.7.jar:2.6.7]
	at inc.evil.spring.tx.management.SpringDeclarativeTxManagementApplication.main(SpringDeclarativeTxManagementApplication.java:30) ~[classes/:na]
Caused by: java.lang.reflect.UndeclaredThrowableException: TransactionCallback threw undeclared checked exception
	at org.springframework.transaction.support.TransactionTemplate.execute(TransactionTemplate.java:150) ~[spring-tx-5.3.19.jar:5.3.19]
	at inc.evil.spring.tx.management.MovieService.saveMovie(SpringDeclarativeTxManagementApplication.java:135) ~[classes/:na]
	at inc.evil.spring.tx.management.SpringDeclarativeTxManagementApplication.lambda$commandLineRunner$0(SpringDeclarativeTxManagementApplication.java:36) ~[classes/:na]
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:777) ~[spring-boot-2.6.7.jar:2.6.7]
	... 5 common frames omitted
Caused by: java.io.IOException: null
	at inc.evil.spring.tx.management.MovieService.offendingMethod(SpringDeclarativeTxManagementApplication.java:150) ~[classes/:na]
	at inc.evil.spring.tx.management.MovieService$1.doInTransaction(SpringDeclarativeTxManagementApplication.java:143) ~[classes/:na]
	at inc.evil.spring.tx.management.MovieService$1.doInTransaction(SpringDeclarativeTxManagementApplication.java:135) ~[classes/:na]
	at org.springframework.transaction.support.TransactionTemplate.execute(TransactionTemplate.java:140) ~[spring-tx-5.3.19.jar:5.3.19]
	... 8 common frames omitted

2022-05-12 10:44:22.573  INFO 1030147 --- [main] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
2022-05-12 10:44:22.574  INFO 1030147 --- [main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...
2022-05-12 10:44:22.575  INFO 1030147 --- [main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.

Process finished with exit code 1

```

That's an implementation detail of `TransactionTemplate`, it rollbacks the transaction on unchecked exceptions (and `Error`), but in the case of checked exceptions, it doesn't commit the transaction (as opposed to declarative transaction management), see the implementation below:

```java
package org.springframework.transaction.support;

public class TransactionTemplate extends DefaultTransactionDefinition implements TransactionOperations, InitializingBean {
    //...
    @Nullable
    public <T> T execute(TransactionCallback<T> action) throws TransactionException {
        Assert.state(this.transactionManager != null, "No PlatformTransactionManager set");
        if (this.transactionManager instanceof CallbackPreferringPlatformTransactionManager) {
            return ((CallbackPreferringPlatformTransactionManager)this.transactionManager).execute(this, action);
        } else {
            TransactionStatus status = this.transactionManager.getTransaction(this);

            Object result;
            try {
                result = action.doInTransaction(status);
            } catch (Error | RuntimeException var5) {
                this.rollbackOnException(status, var5);
                throw var5;
            } catch (Throwable var6) {
                this.rollbackOnException(status, var6);
                throw new UndeclaredThrowableException(var6, "TransactionCallback threw undeclared checked exception");
            }

            this.transactionManager.commit(status);
            return result;
        }
    }
    //...
```

This effectively means that the declarative transaction management does not have the same behavior as the programmatic one. The delarative approach (using the `@Transactional` annotation) by default commits the transaction in case of checked exceptions but the programmatic one,
prevents us from using checked-exceptions in the first place (because of the `TransactionCallback` interface), but even if we try to fool the compiler with the Lombok's `@SneakyThrows` annotation, the transaction still won't be committed.

Also isn't it a bit puzzling that with Lombok's `@SneakyThrows` annotation we were able to throw a checked-exception without declaring it in the method signature? Indeed it is.

Lombok uses a generics trick to fool the compiler. Try out the following example:

```java
public class SayWhaaat {
    public static void main(String[] args) {
        doThrow(new IOException());
    }
 
    static void doThrow(Exception e) {
      SayWhaaat.<RuntimeException> doThrow0(e);
    }
 
    @SuppressWarnings("unchecked")
    static <E extends Exception> void doThrow0(Exception e) throws E {
        throw (E) e;
    }
}
```

The code above compiles just fine and it throws an `IOException` (which is a checked exception), without declaring it! Lombok effectively does something similar with the `@SneakyThrows` annotation.

## Transaction propagation

What will happen if we'll call one `@Transactional` method from another? Will we have 2 transactions or just one? Let's run the example below and look at the logs.

We have 2 services: `MovieService` and `MovieServiceTwo`. First we call the transactional `MovieService.saveMovie()` method and then it tries to invoke the `MovieServiceTwo.save()` which is also transactional.

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final MovieServiceTwo movieServiceTwo;

    public MovieService(MovieServiceTwo movieServiceTwo) {
        this.movieServiceTwo = movieServiceTwo;
    }

    @Transactional
    public Movie saveMovie(String movieName) {
        log.debug("Inside MovieService");
        Movie movie = Movie.builder()
                .name(movieName)
                .build();
        return movieServiceTwo.save(movie);
    }
}

@Service
@Slf4j
class MovieServiceTwo {
    private final EntityManager entityManager;

    public MovieServiceTwo(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional
    public Movie save(Movie movie) {
        log.debug("Inside MovieServiceTwo");
        entityManager.persist(movie);
        return movie;
    }
}

```

If we look at the logs, we can see that we have a single transaction:

```
2022-05-11 15:34:42.455 DEBUG 690207 --- [main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-11 15:34:42.455 DEBUG 690207 --- [main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(2020226167<open>)] for JPA transaction
2022-05-11 15:34:42.457 DEBUG 690207 --- [main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@46b6701e]
2022-05-11 15:34:42.460 DEBUG 690207 --- [main] i.e.s.tx.management.MovieService      : Inside MovieService
2022-05-11 15:34:42.460 DEBUG 690207 --- [main] o.s.orm.jpa.JpaTransactionManager        : Found thread-bound EntityManager [SessionImpl(2020226167<open>)] for JPA transaction
2022-05-11 15:34:42.460 DEBUG 690207 --- [main] o.s.orm.jpa.JpaTransactionManager        : Participating in existing transaction
2022-05-11 15:34:42.463 DEBUG 690207 --- [main] i.e.s.tx.management.MovieServiceTwo      : Inside MovieServiceTwo
2022-05-11 15:34:42.468 DEBUG 690207 --- [main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction commit
2022-05-11 15:34:42.468 DEBUG 690207 --- [main] o.s.orm.jpa.JpaTransactionManager        : Committing JPA transaction on EntityManager [SessionImpl(2020226167<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-11 15:34:42.481 DEBUG 690207 --- [main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(2020226167<open>)] after transaction
```

What happened is that the proxy for `MovieServiceTwo` has found an existing transaction for the current thread and it joined it.

Basically transactions are like a viral disease, they can infect other methods.

The `@Transactional` annotation has an attribute called `propagation` and it specifies the desired behavior when a transactional method is called with or without an active transaction. 

Here are the possible `propagation` levels:
- `@Transactional(propagation = Propagation.REQUIRED)`: the default. Starts a transaction if we don't have one, otherwise join the existing one
- `@Transactional(propagation = Propagation.REQUIRES_NEW)`: Starts a transaction if we don't have one, otherwise suspend the existing one and create a new one
- `@Transactional(propagation = Propagation.SUPPORTS)`: Join the existing transaction. If we don't have a transaction execute without it
- `@Transactional(propagation = Propagation.MANDATORY)`: Join the existing transaction, If  we don't have one, throw an exception
- `@Transactional(propagation = Propagation.NEVER)`: Throw an exception of an active transaction exists. If we don't have one, execute without a transaction
- `@Transactional(propagation = Propagation.NOT_SUPPORTED)`: Suspend the transaction and execute non-transactionally, otherwise execute without a transaction
- `@Transactional(propagation = Propagation.NESTED)`: resembles `Propagation.REQUIRES_NEW`. Starts a transaction if we don't have one, otherwise join the existing one but create a jdbc savepoint beforehand. In case of exceptions, we'll rollback to the savepoint

Let's take a closer look at the transaction propagation level behaviors.

### `@Transactional(propagation = Propagation.REQUIRED)`

When calling a method annotated with `@Transactional(propagation = Propagation.REQUIRED)`, if the current thread is not associated with a transaction, a new transaction will be created. If a transaction exists, the method will just join it.

But what will happen if one `@Transactional` method calls another `@Transactional` method from a different service, with `propagation = Propagation.REQUIRED` and the second service throws an exception?

Do we lose all the work done by the first service as well or only the work done by the second service? Let's have a look:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
  private final ActorService actorService;
  private final EntityManager entityManager;

  public MovieService(ActorService actorService, EntityManager entityManager) {
    this.actorService = actorService;
    this.entityManager = entityManager;
  }

  @Transactional(propagation = Propagation.REQUIRED)
  public Movie saveMovie(String movieName) {
    try {
      Movie movie = Movie.builder()
              .name(movieName)
              .build();
      entityManager.persist(movie);
      log.debug("Calling ActorService");
      actorService.saveActor("John Travolta");
      return movie;
    } catch (Exception e) {
      log.error("Caught an exception");
      return null;
    }
  }
}

@Service
@Slf4j
class ActorService {
  private final EntityManager entityManager;

  public ActorService(EntityManager entityManager) {
    this.entityManager = entityManager;
  }

  @Transactional(propagation = Propagation.REQUIRED)
  public Actor saveActor(String name) {
    Actor actor = Actor.builder()
            .name(name)
            .build();
    entityManager.persist(actor);
    throw new NullPointerException();
  }
}

```

If we look at the logs, we can see that the `ActorService.saveActor()` method has joined the existing transaction (the `Participating in existing transaction`) and because it threw an exception, the transaction was marked as rollback-only by the proxy.

Because the transaction was marked as rollback-only, it cannot be committed, even though it tried to (see `Initiating transaction commit` in the logs). Also we can notice that trying to catch the exception didn't help at all, since the exception "passed though" the proxy and because of that
, it has set the `rollback-only` flag.

We can also observe that all of the database work done by both services was lost (or rolled-back) since we don't have any SQL statements
logged (we've configured `Hibernate` to do so).

```
2022-05-11 17:02:05.767 DEBUG 716638 --- [main] o.s.orm.jpa.JpaTransactionManager  : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-11 17:02:05.767 DEBUG 716638 --- [main] o.s.orm.jpa.JpaTransactionManager  : Opened new EntityManager [SessionImpl(1478683866<open>)] for JPA transaction
2022-05-11 17:02:05.769 DEBUG 716638 --- [main] o.s.orm.jpa.JpaTransactionManager  : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@3ee6dc82]
2022-05-11 17:02:05.777 DEBUG 716638 --- [main] i.e.s.tx.management.MovieService: Calling ActorService
2022-05-11 17:02:05.778 DEBUG 716638 --- [main] o.s.orm.jpa.JpaTransactionManager  : Found thread-bound EntityManager [SessionImpl(1478683866<open>)] for JPA transaction
2022-05-11 17:02:05.778 DEBUG 716638 --- [main] o.s.orm.jpa.JpaTransactionManager  : Participating in existing transaction
2022-05-11 17:02:05.781 DEBUG 716638 --- [main] o.s.orm.jpa.JpaTransactionManager  : Participating transaction failed - marking existing transaction as rollback-only
2022-05-11 17:02:05.781 DEBUG 716638 --- [main] o.s.orm.jpa.JpaTransactionManager  : Setting JPA transaction on EntityManager [SessionImpl(1478683866<open>)] rollback-only
2022-05-11 17:02:05.781 ERROR 716638 --- [main] i.e.s.tx.management.MovieService: Caught an exception
2022-05-11 17:02:05.781 DEBUG 716638 --- [main] o.s.orm.jpa.JpaTransactionManager  : Initiating transaction commit
2022-05-11 17:02:05.781 DEBUG 716638 --- [main] o.s.orm.jpa.JpaTransactionManager  : Committing JPA transaction on EntityManager [SessionImpl(1478683866<open>)]
2022-05-11 17:02:05.782 DEBUG 716638 --- [main] o.s.orm.jpa.JpaTransactionManager  : Closing JPA EntityManager [SessionImpl(1478683866<open>)] after transaction
```

### `@Transactional(propagation = Propagation.REQUIRES_NEW)`

When calling a method annotated with `@Transactional(propagation = Propagation.REQUIRES_NEW)`, if the current thread is not associated with a transaction, a new transaction will be created. 
If a transaction exists, the existing transaction will be suspended and we'll create a new one, execute it and then finally we'll resume the initial one.

But what will happen if one `@Transactional` method calls another `@Transactional(propagation = Propagation.REQUIRES_NEW)` method from a different service, and the second service throws an exception?

In this case we have 2 transactions and we expect a rollback only for the second transaction. Let's have a look:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final ActorService actorService;
    private final EntityManager entityManager;

    public MovieService(ActorService actorService, EntityManager entityManager) {
        this.actorService = actorService;
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public Movie saveMovie(String movieName) {
        try {
            Movie movie = Movie.builder()
                    .name(movieName)
                    .build();
            entityManager.persist(movie);
            log.debug("Calling ActorService");
          actorService.saveActor("John Travolta");
          return movie;
        } catch (Exception e) {
            log.error("Caught an exception");
            return null;
        }
    }
}

@Service
@Slf4j
class ActorService {
    private final EntityManager entityManager;

    public ActorService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public Actor saveActor(String name) {
        Actor actor = Actor.builder()
                .name(name)
                .build();
        entityManager.persist(actor);
        throw new NullPointerException();
    }
}

```

Looking at the logs we can observe that when the `ActorService.saveActor()` method is called, the existing transaction is indeed suspended (`Suspending current transaction, creating new transaction with name [inc.evil.spring.tx.management.ActorService.saveActor]`).

When the `ActorService.saveActor()` throws an exception, the new transaction is rollbacked (`Rolling back JPA transaction on EntityManager [SessionImpl(1959219756<open>)]`) and then the initial transaction is resumed (`Resuming suspended transaction after completion of inner transaction`).

Since the `MovieService.saveMovie()` catches the exception thrown by the `ActorService.saveActor()`, no exceptions will "pass though" the proxy of the `MovieService` class and that means that the initial transaction will be committed (see the logs below).
If we were to remove the `try-catch` block from the `MovieService.saveMovie()` method, both transactions will be rolled-back.

We can also observe that the work done by the `MovieService.saveMovie()` is preserved, we have an insert SQL statement in the logs.

```
2022-05-11 17:19:29.308 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-11 17:19:29.308 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Opened new EntityManager [SessionImpl(1598068850<open>)] for JPA transaction
2022-05-11 17:19:29.310 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@506aa618]
2022-05-11 17:19:29.318 DEBUG 721804 --- [main] i.e.s.tx.management.MovieService: Calling ActorService
2022-05-11 17:19:29.318 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Found thread-bound EntityManager [SessionImpl(1598068850<open>)] for JPA transaction
2022-05-11 17:19:29.319 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Suspending current transaction, creating new transaction with name [inc.evil.spring.tx.management.ActorService.saveActor]
2022-05-11 17:19:29.319 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Opened new EntityManager [SessionImpl(1959219756<open>)] for JPA transaction
2022-05-11 17:19:29.319 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@77c23d90]
2022-05-11 17:19:29.321 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Initiating transaction rollback
2022-05-11 17:19:29.322 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Rolling back JPA transaction on EntityManager [SessionImpl(1959219756<open>)]
2022-05-11 17:19:29.322 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Closing JPA EntityManager [SessionImpl(1959219756<open>)] after transaction
2022-05-11 17:19:29.322 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Resuming suspended transaction after completion of inner transaction
2022-05-11 17:19:29.322 ERROR 721804 --- [main] i.e.s.tx.management.MovieService: Caught an exception
2022-05-11 17:19:29.322 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Initiating transaction commit
2022-05-11 17:19:29.322 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Committing JPA transaction on EntityManager [SessionImpl(1598068850<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-11 17:19:29.330 DEBUG 721804 --- [main] o.s.orm.jpa.JpaTransactionManager  : Closing JPA EntityManager [SessionImpl(1598068850<open>)] after transaction
```

### `@Transactional(propagation = Propagation.NESTED)`

The `propagation = Propagation.NESTED` works pretty much the same way as `propagation = Propagation.REQUIRES_NEW`,  if the current thread is not associated with a transaction, a new transaction will be created.
If a transaction exists, we'll create a `JDBC Savepoint` before entering the new transactional method and in case of failure we will rollback to the jdbc savepoint.

Basically if one `@Transactional` method calls another `@Transactional(propagation = Propagation.NESTED)` method from a different service, and the second service throws an exception, we will rollback to the jdbc savepoint and in this way we'll preserve the work done by the first service.
Let's have a look:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final ActorService actorService;
    private final EntityManager entityManager;

    public MovieService(ActorService actorService, EntityManager entityManager) {
        this.actorService = actorService;
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public Movie saveMovie(String movieName) {
        try {
            Movie movie = Movie.builder()
                    .name(movieName)
                    .build();
            entityManager.persist(movie);
            log.debug("Calling ActorService");
          actorService.saveActor("John Travolta");
          return movie;
        } catch (Exception e) {
            log.error("Caught an exception");
            return null;
        }
    }
}

@Service
@Slf4j
class ActorService {
    private final EntityManager entityManager;

    public ActorService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.NESTED)
    public Actor saveActor(String name) {
        Actor actor = Actor.builder()
                .name(name)
                .build();
        entityManager.persist(actor);
        throw new NullPointerException();
    }
}

```

```
2022-05-11 17:43:55.526 DEBUG 729067 --- [main] o.s.orm.jpa.JpaTransactionManager  : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-11 17:43:55.527 DEBUG 729067 --- [main] o.s.orm.jpa.JpaTransactionManager  : Opened new EntityManager [SessionImpl(711964207<open>)] for JPA transaction
2022-05-11 17:43:55.528 DEBUG 729067 --- [main] o.s.orm.jpa.JpaTransactionManager  : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@7e2bd5e6]
2022-05-11 17:43:55.536 DEBUG 729067 --- [main] i.e.s.tx.management.MovieService: Calling ActorService
2022-05-11 17:43:55.537 DEBUG 729067 --- [main] o.s.orm.jpa.JpaTransactionManager  : Found thread-bound EntityManager [SessionImpl(711964207<open>)] for JPA transaction
2022-05-11 17:43:55.537 DEBUG 729067 --- [main] o.s.orm.jpa.JpaTransactionManager  : Creating nested transaction with name [inc.evil.spring.tx.management.ActorService.saveActor]
2022-05-11 17:43:55.537 ERROR 729067 --- [main] i.e.s.tx.management.MovieService: Caught an exception
2022-05-11 17:43:55.537 DEBUG 729067 --- [main] o.s.orm.jpa.JpaTransactionManager  : Initiating transaction commit
2022-05-11 17:43:55.537 DEBUG 729067 --- [main] o.s.orm.jpa.JpaTransactionManager  : Committing JPA transaction on EntityManager [SessionImpl(711964207<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-11 17:43:55.545 DEBUG 729067 --- [main] o.s.orm.jpa.JpaTransactionManager  : Closing JPA EntityManager [SessionImpl(711964207<open>)] after transaction
```

Looking at the logs we can see that only the work done by the `MovieService.saveMovie()` method was successfully inserted into the database.

### `@Transactional(propagation = Propagation.SUPPORTS)`

This is an easy one, it works as if the `@Transactional` annotation is not present. If we have an active transaction, we join it. If we don't, execute without a transaction.

Let's take a look at the example below:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final ActorService actorService;
    private final EntityManager entityManager;

    public MovieService(ActorService actorService, EntityManager entityManager) {
        this.actorService = actorService;
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public Movie saveMovie(String movieName) {
        try {
            Movie movie = Movie.builder()
                    .name(movieName)
                    .build();
            entityManager.persist(movie);
            log.debug("Calling ActorService");
            actorService.saveActor("John Travolta");
            return movie;
        } catch (Exception e) {
            log.error("Caught an exception");
            return null;
        }
    }
}

@Service
@Slf4j
class ActorService {
    private final EntityManager entityManager;

    public ActorService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.SUPPORTS)
    public Actor saveActor(String name) {
        Actor actor = Actor.builder()
                .name(name)
                .build();
        entityManager.persist(actor);
        return actor;
    }
}
```

We can see that the `ActorService.saveActor` which has `@Transactional(propagation = Propagation.SUPPORTS)` has joined the existing transaction, but if the `@Transactional` annotation wasn't present, the same thing would happen.

```
2022-05-12 10:01:38.812 DEBUG 1017539 --- [main] o.s.orm.jpa.JpaTransactionManager  : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-12 10:01:38.812 DEBUG 1017539 --- [main] o.s.orm.jpa.JpaTransactionManager  : Opened new EntityManager [SessionImpl(1185631996<open>)] for JPA transaction
2022-05-12 10:01:38.814 DEBUG 1017539 --- [main] o.s.orm.jpa.JpaTransactionManager  : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@40016ce1]
2022-05-12 10:01:38.822 DEBUG 1017539 --- [main] i.e.s.tx.management.MovieService: Calling ActorService
2022-05-12 10:01:38.822 DEBUG 1017539 --- [main] o.s.orm.jpa.JpaTransactionManager  : Found thread-bound EntityManager [SessionImpl(1185631996<open>)] for JPA transaction
2022-05-12 10:01:38.822 DEBUG 1017539 --- [main] o.s.orm.jpa.JpaTransactionManager  : Participating in existing transaction
2022-05-12 10:01:38.824 DEBUG 1017539 --- [main] o.s.orm.jpa.JpaTransactionManager  : Initiating transaction commit
2022-05-12 10:01:38.824 DEBUG 1017539 --- [main] o.s.orm.jpa.JpaTransactionManager  : Committing JPA transaction on EntityManager [SessionImpl(1185631996<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
Hibernate: 
    insert 
    into
        actors
        (name, id) 
    values
        (?, ?)
2022-05-12 10:01:38.832 DEBUG 1017539 --- [main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(1185631996<open>)] after transaction
```

What will happen if both methods have the `@Transactional(propagation = Propagation.SUPPORTS)` annotation? Let's have a look:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
  private final ActorService actorService;
  private final EntityManager entityManager;

  public MovieService(ActorService actorService, EntityManager entityManager) {
    this.actorService = actorService;
    this.entityManager = entityManager;
  }

  @Transactional(propagation = Propagation.SUPPORTS)
  public Movie saveMovie(String movieName) {
    Movie movie = Movie.builder()
            .name(movieName)
            .build();
    entityManager.persist(movie);
    log.debug("Calling ActorService");
    actorService.saveActor("John Travolta");
    return movie;
  }
}

@Service
@Slf4j
class ActorService {
  private final EntityManager entityManager;

  public ActorService(EntityManager entityManager) {
    this.entityManager = entityManager;
  }

  @Transactional(propagation = Propagation.SUPPORTS)
  public Actor saveActor(String name) {
    Actor actor = Actor.builder()
            .name(name)
            .build();
    entityManager.persist(actor);
    return actor;
  }
}


```

In this case no transaction will be created and we'll get a `TransactionRequiredException` since `Hibernate` requires a transaction for the `EntityManager.persist()` method to execute successfully (see logs below):

```
2022-05-12 10:00:19.245 DEBUG 1017050 --- [main] o.s.orm.jpa.EntityManagerFactoryUtils    : Opening JPA EntityManager
2022-05-12 10:00:19.246 DEBUG 1017050 --- [main] o.s.orm.jpa.JpaTransactionManager        : Should roll back transaction but cannot - no transaction available
2022-05-12 10:00:19.247  INFO 1017050 --- [main] ConditionEvaluationReportLoggingListener : 

Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2022-05-12 10:00:19.256 ERROR 1017050 --- [main] o.s.boot.SpringApplication               : Application run failed

java.lang.IllegalStateException: Failed to execute CommandLineRunner
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:780) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.callRunners(SpringApplication.java:761) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:310) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1312) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1301) ~[spring-boot-2.6.7.jar:2.6.7]
	at inc.evil.spring.tx.management.SpringDeclarativeTxManagementApplication.main(SpringDeclarativeTxManagementApplication.java:27) ~[classes/:na]
Caused by: javax.persistence.TransactionRequiredException: No EntityManager with actual transaction available for current thread - cannot reliably process 'persist' call
```

Also notice the `Should roll back transaction but cannot - no transaction available` in the logs. This means that `@Transactional(propagation = Propagation.SUPPORTS)` is not precisely equivalent to not having the annotation at all. It does have the same effect, but the method is still intercepted
by the proxy and that means that it's a bit slower than not having the annotation at all.

### `@Transactional(propagation = Propagation.NOT_SUPPORTED)`

The `@Transactional(propagation = Propagation.NOT_SUPPORTED)` propagation works in the following way: if we don't have a transaction, no problem, execute without it. If we do have one, suspend it, execute the method with the `propagation = Propagation.NOT_SUPPORTED` and then resume the transaction.

Basically it is useful when we want to call a method from a different service which doesn't need a transaction and at the same time we want to prevent that method from marking the transaction as rollback-only in case of exceptions. Let's have a look:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final ActorService actorService;
    private final EntityManager entityManager;

    public MovieService(ActorService actorService, EntityManager entityManager) {
        this.actorService = actorService;
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public Movie saveMovie(String movieName) {
        Movie movie = Movie.builder()
                .name(movieName)
                .build();
        entityManager.persist(movie);
        log.debug("Calling ActorService");
        try {
            actorService.saveActor("John Travolta");
        } catch (Exception e) {
            log.error("Caught an exception");
        }
        return movie;
    }
}

@Service
@Slf4j
class ActorService {
    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public Actor saveActor(String name) {
        throw new RuntimeException();
    }
}

```

If we analyze the logs we can see that the `ActorService.saveActor()` suspends the transaction (`Suspending current transaction`) and then it resumes it (`Resuming suspended transaction after completion of inner transaction`).

Even though the `ActorService.saveActor()` threw a `RuntimeException` which triggers a rollback usually, we successfully committed the transaction since the exception was thrown when the transaction was suspended (see logs below):

```
2022-05-12 11:18:46.933 DEBUG 1040575 --- [main] o.s.orm.jpa.JpaTransactionManager  : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-12 11:18:46.933 DEBUG 1040575 --- [main] o.s.orm.jpa.JpaTransactionManager  : Opened new EntityManager [SessionImpl(1421763091<open>)] for JPA transaction
2022-05-12 11:18:46.936 DEBUG 1040575 --- [main] o.s.orm.jpa.JpaTransactionManager  : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@7eee6c13]
2022-05-12 11:18:46.947 DEBUG 1040575 --- [main] i.e.s.tx.management.MovieService: Calling ActorService
2022-05-12 11:18:46.948 DEBUG 1040575 --- [main] o.s.orm.jpa.JpaTransactionManager  : Found thread-bound EntityManager [SessionImpl(1421763091<open>)] for JPA transaction
2022-05-12 11:18:46.948 DEBUG 1040575 --- [main] o.s.orm.jpa.JpaTransactionManager  : Suspending current transaction
2022-05-12 11:18:46.951 DEBUG 1040575 --- [main] o.s.orm.jpa.JpaTransactionManager  : Should roll back transaction but cannot - no transaction available
2022-05-12 11:18:46.951 DEBUG 1040575 --- [main] o.s.orm.jpa.JpaTransactionManager  : Resuming suspended transaction after completion of inner transaction
2022-05-12 11:18:46.951 ERROR 1040575 --- [main] i.e.s.tx.management.MovieService: Caught an exception
2022-05-12 11:18:46.951 DEBUG 1040575 --- [main] o.s.orm.jpa.JpaTransactionManager  : Initiating transaction commit
2022-05-12 11:18:46.951 DEBUG 1040575 --- [main] o.s.orm.jpa.JpaTransactionManager  : Committing JPA transaction on EntityManager [SessionImpl(1421763091<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-12 11:18:46.964 DEBUG 1040575 --- [main] o.s.orm.jpa.JpaTransactionManager   : Closing JPA EntityManager [SessionImpl(1421763091<open>)] after transaction
```

### `@Transactional(propagation = Propagation.NEVER)`

This transaction propagation level does not support transactions. If we'll call a method annotated with `@Transactional(propagation = Propagation.NEVER)` without a transaction, everything will work fine. If we do have a transaction, we'll get an exception since the method doesn't support transactions.

This propagation level is useful for scenarios where we have a method which doesn't need a transaction and we want to prevent others from calling the method when there's an active transaction. It is considered that transactions should be as small as possible and we can use this propagation level
to prevent a long-running method being called from a transaction, since that will increase significantly the transaction lifespan. Let's have a look:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final ActorService actorService;
    private final EntityManager entityManager;

    public MovieService(ActorService actorService, EntityManager entityManager) {
        this.actorService = actorService;
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.NEVER)
    public Movie saveMovie(String movieName) {
        return Movie.builder()
                .name(movieName)
                .build();
    }
}
```

If we look at the logs, we can see that no transaction was created:

```
2022-05-12 11:41:01.254  INFO 1047157 --- [main] SpringDeclarativeTxManagementApplication : Started SpringDeclarativeTxManagementApplication in 1.045 seconds (JVM running for 1.338)
2022-05-12 11:41:01.260  INFO 1047157 --- [ionShutdownHook] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
```

Let's try out a different example. What will happen if we call a `@Transactional(propagation = Propagation.NEVER)` with an active transaction, like shown below?

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final ActorService actorService;
    private final EntityManager entityManager;

    public MovieService(ActorService actorService, EntityManager entityManager) {
        this.actorService = actorService;
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public Movie saveMovie(String movieName) {
        Movie movie = Movie.builder()
                .name(movieName)
                .build();
        entityManager.persist(movie);
        log.debug("Calling ActorService");
        actorService.saveActor("John Travolta");
        return movie;
    }
}

@Service
@Slf4j
class ActorService {
    @Transactional(propagation = Propagation.NEVER)
    public Actor saveActor(String name) {
        return Actor.builder()
                .name(name)
                .build();
    }
}

```

In this case a `IllegalTransactionStateException` will be thrown since the `ActorService.saveActor()` method does not support transactions (see logs below):

```
2022-05-12 11:43:01.090 DEBUG 1047930 --- [main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-12 11:43:01.090 DEBUG 1047930 --- [main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(711964207<open>)] for JPA transaction
2022-05-12 11:43:01.091 DEBUG 1047930 --- [main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@7e2bd5e6]
2022-05-12 11:43:01.100 DEBUG 1047930 --- [main] i.e.s.tx.management.MovieService      : Calling ActorService
2022-05-12 11:43:01.101 DEBUG 1047930 --- [main] o.s.orm.jpa.JpaTransactionManager        : Found thread-bound EntityManager [SessionImpl(711964207<open>)] for JPA transaction
2022-05-12 11:43:01.101 DEBUG 1047930 --- [main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction rollback
2022-05-12 11:43:01.101 DEBUG 1047930 --- [main] o.s.orm.jpa.JpaTransactionManager        : Rolling back JPA transaction on EntityManager [SessionImpl(711964207<open>)]
2022-05-12 11:43:01.102 DEBUG 1047930 --- [main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(711964207<open>)] after transaction
2022-05-12 11:43:01.103  INFO 1047930 --- [main] ConditionEvaluationReportLoggingListener : 

Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2022-05-12 11:43:01.111 ERROR 1047930 --- [main] o.s.boot.SpringApplication               : Application run failed

java.lang.IllegalStateException: Failed to execute CommandLineRunner
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:780) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.callRunners(SpringApplication.java:761) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:310) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1312) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1301) ~[spring-boot-2.6.7.jar:2.6.7]
	at inc.evil.spring.tx.management.SpringDeclarativeTxManagementApplication.main(SpringDeclarativeTxManagementApplication.java:30) ~[classes/:na]
Caused by: org.springframework.transaction.IllegalTransactionStateException: Existing transaction found for transaction marked with propagation 'never'
	at org.springframework.transaction.support.AbstractPlatformTransactionManager.handleExistingTransaction(AbstractPlatformTransactionManager.java:413) ~[spring-tx-5.3.19.jar:5.3.19]
```

What if we try to catch the `IllegalTransactionStateException`, will that work (meaning will the transaction started by `MovieService.saveMovie()` method be marked as rollback-only or not)? Let's see:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final ActorService actorService;
    private final EntityManager entityManager;

    public MovieService(ActorService actorService, EntityManager entityManager) {
        this.actorService = actorService;
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public Movie saveMovie(String movieName) {
        Movie movie = Movie.builder()
                .name(movieName)
                .build();
        entityManager.persist(movie);
        log.debug("Calling ActorService");
        try {
            actorService.saveActor("John Travolta");
        } catch (IllegalTransactionStateException e) {
            log.error("Caught an exception");
        }
        return movie;
    }
}

@Service
@Slf4j
class ActorService {
    @Transactional(propagation = Propagation.NEVER)
    public Actor saveActor(String name) {
        return Actor.builder()
                .name(name)
                .build();
    }
}

```

We can see from the logs that the transaction was committed successfully. It seems that the proxy for the `ActorService` class does not mark the transaction as rollback-only in this case.

```
2022-05-12 11:46:32.240 DEBUG 1049093 --- [main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-12 11:46:32.240 DEBUG 1049093 --- [main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(1478683866<open>)] for JPA transaction
2022-05-12 11:46:32.242 DEBUG 1049093 --- [main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@3ee6dc82]
2022-05-12 11:46:32.251 DEBUG 1049093 --- [main] i.e.s.tx.management.MovieService      : Calling ActorService
2022-05-12 11:46:32.251 DEBUG 1049093 --- [main] o.s.orm.jpa.JpaTransactionManager        : Found thread-bound EntityManager [SessionImpl(1478683866<open>)] for JPA transaction
2022-05-12 11:46:32.251 ERROR 1049093 --- [main] i.e.s.tx.management.MovieService      : Caught an exception
2022-05-12 11:46:32.251 DEBUG 1049093 --- [main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction commit
2022-05-12 11:46:32.251 DEBUG 1049093 --- [main] o.s.orm.jpa.JpaTransactionManager        : Committing JPA transaction on EntityManager [SessionImpl(1478683866<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-12 11:46:32.258 DEBUG 1049093 --- [main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(1478683866<open>)] after transaction
```

### `@Transactional(propagation = Propagation.MANDATORY)`

The `@Transactional(propagation = Propagation.MANDATORY)` is the opposite of `Propagation.NEVER`. In order to call a method with the `@Transactional(propagation = Propagation.MANDATORY)` we need to have an active transaction, otherwise we'll get an exception.

Let's have a closer look:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final ActorService actorService;
    private final EntityManager entityManager;

    public MovieService(ActorService actorService, EntityManager entityManager) {
        this.actorService = actorService;
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public Movie saveMovie(String movieName) {
        Movie movie = Movie.builder()
                .name(movieName)
                .build();
        entityManager.persist(movie);
        log.debug("Calling ActorService");
        actorService.saveActor("John Travolta");
        return movie;
    }
}

@Service
@Slf4j
class ActorService {
    private final EntityManager entityManager;

    public ActorService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.MANDATORY)
    public Actor saveActor(String name) {
        Actor actor = Actor.builder()
                .name(name)
                .build();
        entityManager.persist(actor);
        return actor;
    }
}

```

We can see from the logs that everything worked fine, we have a single transaction (started by `MovieService.saveMovie()`) and the `ActorService.saveActor()` has joined it:

```
2022-05-12 11:52:22.465 DEBUG 1050965 --- [main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-12 11:52:22.465 DEBUG 1050965 --- [main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(1186328673<open>)] for JPA transaction
2022-05-12 11:52:22.466 DEBUG 1050965 --- [main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@376af784]
2022-05-12 11:52:22.475 DEBUG 1050965 --- [main] i.e.s.tx.management.MovieService      : Calling ActorService
2022-05-12 11:52:22.475 DEBUG 1050965 --- [main] o.s.orm.jpa.JpaTransactionManager        : Found thread-bound EntityManager [SessionImpl(1186328673<open>)] for JPA transaction
2022-05-12 11:52:22.475 DEBUG 1050965 --- [main] o.s.orm.jpa.JpaTransactionManager        : Participating in existing transaction
2022-05-12 11:52:22.477 DEBUG 1050965 --- [main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction commit
2022-05-12 11:52:22.478 DEBUG 1050965 --- [main] o.s.orm.jpa.JpaTransactionManager        : Committing JPA transaction on EntityManager [SessionImpl(1186328673<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
Hibernate: 
    insert 
    into
        actors
        (name, id) 
    values
        (?, ?)
2022-05-12 11:52:22.491 DEBUG 1050965 --- [main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(1186328673<open>)] after transaction
2022-05-12 11:52:22.494  INFO 1050965 --- [ionShutdownHook] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
2022-05-12 11:52:22.495  INFO 1050965 --- [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...
2022-05-12 11:52:22.497  INFO 1050965 --- [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.

Process finished with exit code 0

```

If we try to call a method annotated with `@Transactional(propagation = Propagation.MANDATORY)` without a transaction, we'll get an exception:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final EntityManager entityManager;

    public MovieService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.MANDATORY)
    public Movie saveMovie(String movieName) {
        Movie movie = Movie.builder()
                .name(movieName)
                .build();
        entityManager.persist(movie);
        return movie;
    }
}

```

```
2022-05-12 11:54:20.714  INFO 1051591 --- [main] ConditionEvaluationReportLoggingListener : 

Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2022-05-12 11:54:20.722 ERROR 1051591 --- [main] o.s.boot.SpringApplication               : Application run failed

java.lang.IllegalStateException: Failed to execute CommandLineRunner
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:780) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.callRunners(SpringApplication.java:761) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:310) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1312) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1301) ~[spring-boot-2.6.7.jar:2.6.7]
	at inc.evil.spring.tx.management.SpringDeclarativeTxManagementApplication.main(SpringDeclarativeTxManagementApplication.java:30) ~[classes/:na]
Caused by: org.springframework.transaction.IllegalTransactionStateException: No existing transaction found for transaction marked with propagation 'mandatory'
	at org.springframework.transaction.support.AbstractPlatformTransactionManager.getTransaction(AbstractPlatformTransactionManager.java:362) ~[spring-tx-5.3.19.jar:5.3.19]
```

## Transaction timeouts

The `@Transactional` annotation has an attribute called `timeout` which specifies the transaction timeout in seconds. If the transaction won't be committed in the given number of seconds, it will be automatically rolled-back.

The `timeout` attribute along with `propagation = Propagation.REQUIRED` sometimes has interesting behavior. When a method with `propagation = Propagation.REQUIRED` is joining an existing transaction, 
it inherits the transaction attributes like `isolation`, `readOnly` and `timeout` from the existing transaction. This could lead to unexpected results sometimes.

Let's look at an example. The `MovieService.saveMovie()` method is the one starting the transaction and it declares the transaction `timeout` as one second. The `ActorService.saveActor()` method will join the existing transaction, but it wants a timeout of 10 seconds.
Also the `ActorService.saveActor()` has an artificial delay of 2 seconds. In this case, what do you think will happen?

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final ActorService actorService;
    private final EntityManager entityManager;

    public MovieService(ActorService actorService, EntityManager entityManager) {
        this.actorService = actorService;
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.REQUIRED, timeout = 1)
    public Movie saveMovie(String movieName) {
        Movie movie = Movie.builder()
                .name(movieName)
                .build();
        entityManager.persist(movie);
        actorService.saveActor("John Travolta");
        return movie;
    }
}

@Service
@Slf4j
class ActorService {
    @Transactional(propagation = Propagation.REQUIRED, timeout = 10)
    public Actor saveActor(String name) {
        try {
            TimeUnit.SECONDS.sleep(2);
            return null;
        } catch (InterruptedException e) {
            return null;
        }
    }
}

```

Well, if we look at the logs we can see that the transaction timed-out and was rolled-back, which basically means that `ActorService.saveActor()` method has inherited the `timeout` attribute from the existing transaction it has joined. The transaction was rolled-back since the transaction
timeout was one second but the `ActorService.saveActor()` method executed for 2 seconds.

```
2022-05-12 15:18:16.374 DEBUG 1135044 --- [main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,timeout_1
2022-05-12 15:18:16.374 DEBUG 1135044 --- [main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(337295973<open>)] for JPA transaction
2022-05-12 15:18:16.376 DEBUG 1135044 --- [main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@d504137]
2022-05-12 15:18:16.384 DEBUG 1135044 --- [main] o.s.orm.jpa.JpaTransactionManager        : Found thread-bound EntityManager [SessionImpl(337295973<open>)] for JPA transaction
2022-05-12 15:18:16.385 DEBUG 1135044 --- [main] o.s.orm.jpa.JpaTransactionManager        : Participating in existing transaction
2022-05-12 15:18:18.391 DEBUG 1135044 --- [main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction commit
2022-05-12 15:18:18.392 DEBUG 1135044 --- [main] o.s.orm.jpa.JpaTransactionManager        : Committing JPA transaction on EntityManager [SessionImpl(337295973<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-12 15:18:18.405 DEBUG 1135044 --- [main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction rollback after commit exception

org.springframework.orm.jpa.JpaSystemException: transaction timeout expired; nested exception is org.hibernate.TransactionException: transaction timeout expired
	at org.springframework.orm.jpa.vendor.HibernateJpaDialect.convertHibernateAccessException(HibernateJpaDialect.java:331) ~[spring-orm-5.3.19.jar:5.3.19]
	at org.springframework.orm.jpa.vendor.HibernateJpaDialect.translateExceptionIfPossible(HibernateJpaDialect.java:233) ~[spring-orm-5.3.19.jar:5.3.19]

```

Apart from `timeout`, we mentioned that the `isolation` and `readOnly` attributes of the `@Transactional` annotation behave in the same way. In case of joining an existing transaction these attributes are inherited from the initial transaction.
It is possible to configure the `PlatformTransactionManager` to throw an exception when we join an existing transaction and there's a mismatch of the `isolation` and the `readOnly` attributes (meaning the values of these attributes for the joining transaction differ from the value
of these atributes of the original transaction). Let's have a look at an example:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }

  @Bean
  @Primary
  public PlatformTransactionManager transactionManager(EntityManagerFactory entityManagerFactory) {
    JpaTransactionManager transactionManager = new JpaTransactionManager(entityManagerFactory);
    transactionManager.setValidateExistingTransaction(true); //throw an exception when joining an existing transaction and there's a mismatch of readOnly and isolation attributes
    return transactionManager;
  }
}


@Service
@Slf4j
class MovieService {
    private final ActorService actorService;
    private final EntityManager entityManager;

    public MovieService(ActorService actorService, EntityManager entityManager) {
        this.actorService = actorService;
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public Movie saveMovie(String movieName) {
        Movie movie = Movie.builder()
                .name(movieName)
                .build();
        entityManager.persist(movie);
        actorService.saveActor("John Travolta");
        return movie;
    }
}

@Service
@Slf4j
class ActorService {
    private final EntityManager entityManager;

    public ActorService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional(propagation = Propagation.REQUIRED, isolation = Isolation.SERIALIZABLE)
    public Actor saveActor(String name) {
        Actor actor = Actor.builder()
                .name(name)
                .build();
        entityManager.persist(actor);
        return actor;
    }
}
```

In the example above, the `MovieService.saveMovie()` method creates a transaction. Since it didn't specify any isolation level, the default is used. It depends on the database we're using what isolation level will be used. Since we're using `PostgreSQL`, the default
isolation level will be `READ_COMMITTED`.

The `ActorService.saveActor()` tries to join the existing transaction, but with a different isolation level (`Isolation.SERIALIZABLE`). By default the joining transaction will inherit the `isolation` attribute of the original transaction (meaning that the `Isolation.SERIALIZABLE` will be
ignored).

But since we've configured the `PlatformTransactionManager` and we've configured the `validateExistingTransaction` property and set it to `true`, we should get an exception since there's a mismatch of isolation levels. Let's see if that't the case:

```
2022-05-14 01:55:01.750 DEBUG 1372400 --- [main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-14 01:55:01.751 DEBUG 1372400 --- [main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(612089786<open>)] for JPA transaction
2022-05-14 01:55:01.752 DEBUG 1372400 --- [main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@2bfa17b0]
2022-05-14 01:55:01.760 DEBUG 1372400 --- [main] o.s.orm.jpa.JpaTransactionManager        : Found thread-bound EntityManager [SessionImpl(612089786<open>)] for JPA transaction
2022-05-14 01:55:01.760 DEBUG 1372400 --- [main] o.s.orm.jpa.JpaTransactionManager        : Participating in existing transaction
2022-05-14 01:55:01.761 DEBUG 1372400 --- [main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction rollback
2022-05-14 01:55:01.761 DEBUG 1372400 --- [main] o.s.orm.jpa.JpaTransactionManager        : Rolling back JPA transaction on EntityManager [SessionImpl(612089786<open>)]
2022-05-14 01:55:01.761 DEBUG 1372400 --- [main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(612089786<open>)] after transaction
2022-05-14 01:55:01.762  INFO 1372400 --- [main] ConditionEvaluationReportLoggingListener : 

Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2022-05-14 01:55:01.769 ERROR 1372400 --- [main] o.s.boot.SpringApplication               : Application run failed

java.lang.IllegalStateException: Failed to execute CommandLineRunner
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:780) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.callRunners(SpringApplication.java:761) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:310) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1312) ~[spring-boot-2.6.7.jar:2.6.7]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1301) ~[spring-boot-2.6.7.jar:2.6.7]
	at inc.evil.spring.tx.management.SpringDeclarativeTxManagementApplication.main(SpringDeclarativeTxManagementApplication.java:34) ~[classes/:na]
Caused by: org.springframework.transaction.IllegalTransactionStateException: Participating transaction with definition [PROPAGATION_REQUIRED,ISOLATION_SERIALIZABLE] specifies isolation level which is incompatible with existing transaction: (unknown)
```

Indeed we've got a `IllegalTransactionStateException`, stating the incompatible transaction isolation levels.

## Read-only transactions

When using the `JpaTransactionManager` (configured by default when using `spring-data-jpa`), every transaction which is created also creates an `EntityManager` which represents the so-called "unit of work" in `Hibernate`.
Though the `EntityManager` is not thread-safe, it's not a problem since transactions are thread-local (or bound to a specific thread).

We can easily verify that when creating a new transaction an `EntityManager` is created as well by looking at the following example:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.saveMovie("Pulp fiction");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final EntityManager entityManager;

    public MovieServiceOne(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional
    protected Movie saveMovie(String movieName) {
        Movie movie = Movie.builder()
                .name(movieName)
                .build();
        entityManager.persist(movie);
        return movie;
    }
}

```

In the logs we can observe the sequence: `Opened new EntityManager [SessionImpl(1982072255<open>)] for JPA transaction` and `Closing JPA EntityManager [SessionImpl(1982072255<open>)] after transaction`.

```
2022-05-13 08:42:02.640 DEBUG 1250340 --- [main] o.s.orm.jpa.JpaTransactionManager: Creating new transaction with name [inc.evil.spring.tx.management.MovieServiceOne.saveMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-13 08:42:02.641 DEBUG 1250340 --- [main] o.s.orm.jpa.JpaTransactionManager: Opened new EntityManager [SessionImpl(1982072255<open>)] for JPA transaction
2022-05-13 08:42:02.642 DEBUG 1250340 --- [main] o.s.orm.jpa.JpaTransactionManager: Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@37a67cf]
2022-05-13 08:42:02.652 DEBUG 1250340 --- [main] o.s.orm.jpa.JpaTransactionManager: Initiating transaction commit
2022-05-13 08:42:02.653 DEBUG 1250340 --- [main] o.s.orm.jpa.JpaTransactionManager: Committing JPA transaction on EntityManager [SessionImpl(1982072255<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-13 08:42:02.665 DEBUG 1250340 --- [main] o.s.orm.jpa.JpaTransactionManager: Closing JPA EntityManager [SessionImpl(1982072255<open>)] after transaction
```

It is known that the `EntityManager` acts as a first-level cache (in different sources there's different nomenclature for this, sometimes this is called that the `EntityManager` has a `persistence context` or that the `EntityManager` represents the `persistence context`), which 
basically is a cache for entities in the `persistent` state.

The question which arrives at this point is: when a transactional method from one service joins an existing transaction (created by another service), is the `EntityManager` reused? Let's have a look at an example:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieServiceOne movieService) {
    return args -> {
      movieService.findMovie("aa3e4567-e89b-12d3-b457-5267141750aa");
    };
  }
}

@Service
@Slf4j
class MovieServiceOne {
    private final EntityManager entityManager;
    private final MovieServiceTwo movieServiceTwo;

    public MovieServiceOne(EntityManager entityManager, MovieServiceTwo movieServiceTwo) {
        this.entityManager = entityManager;
        this.movieServiceTwo = movieServiceTwo;
    }

    @Transactional
    protected Movie findMovie(String id) {
        Movie movie = entityManager.find(Movie.class, id);
        log.debug("Found movie: {}", movie);
        movieServiceTwo.findMovie(id);
        return movie;
    }
}

@Service
@Slf4j
class MovieServiceTwo {
    private final EntityManager entityManager;

    public MovieServiceTwo(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional
    protected Movie findMovie(String id) {
        Movie movie = entityManager.find(Movie.class, id);
        log.debug("Found movie: {}", movie);
        return movie;
    }
}
```

In the example above we have 2 `@Transactional` methods, one calling the other and since we're using the defaults, a single transaction will be created (since the default propagation level is `PROPAGATION_REQUIRED`). In the both `@Transactional` methods we try to find the same `JPA` entity.
The question is: how many `EntityManager`s will be created and how many SQL statements will be executed? Let's check the logs:

```
2022-05-13 09:02:20.836 DEBUG 1252131 --- [main] o.s.orm.jpa.JpaTransactionManager  : Creating new transaction with name [inc.evil.spring.tx.management.MovieServiceOne.findMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-13 09:02:20.836 DEBUG 1252131 --- [main] o.s.orm.jpa.JpaTransactionManager  : Opened new EntityManager [SessionImpl(1496396949<open>)] for JPA transaction
2022-05-13 09:02:20.837 DEBUG 1252131 --- [main] o.s.orm.jpa.JpaTransactionManager  : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@2eb6d34a]
Hibernate: 
    select
        movie0_.id as id1_1_0_,
        movie0_.name as name2_1_0_ 
    from
        movies movie0_ 
    where
        movie0_.id=?
2022-05-13 09:02:20.849 DEBUG 1252131 --- [main] i.e.s.tx.management.MovieServiceOne: Found movie: Movie{name='Pulp Fiction', id='aa3e4567-e89b-12d3-b457-5267141750aa'}
2022-05-13 09:02:20.850 DEBUG 1252131 --- [main] o.s.orm.jpa.JpaTransactionManager  : Found thread-bound EntityManager [SessionImpl(1496396949<open>)] for JPA transaction
2022-05-13 09:02:20.850 DEBUG 1252131 --- [main] o.s.orm.jpa.JpaTransactionManager  : Participating in existing transaction
2022-05-13 09:02:20.850 DEBUG 1252131 --- [main] i.e.s.tx.management.MovieServiceTwo: Found movie: Movie{name='Pulp Fiction', id='aa3e4567-e89b-12d3-b457-5267141750aa'}
2022-05-13 09:02:20.850 DEBUG 1252131 --- [main] o.s.orm.jpa.JpaTransactionManager  : Initiating transaction commit
2022-05-13 09:02:20.850 DEBUG 1252131 --- [main] o.s.orm.jpa.JpaTransactionManager  : Committing JPA transaction on EntityManager [SessionImpl(1496396949<open>)]
```

From the logs we can see that we have a single `EntityManager`. The `MovieServiceOne.findMovie()` method has created one `Opened new EntityManager [SessionImpl(1496396949<open>)] for JPA transaction` and the `MovieServiceTwo.findMovie()` 
is using the same `EntityManager` (see `Found thread-bound EntityManager [SessionImpl(1496396949<open>)] for JPA transaction` in the logs).

We also can observe that we've executed a single `SQL` select statement, even though we have 2 calls to the `EntityManager.find()` method. The reason is that the `EntityManager.find()` first checks the persistence context to see if the desired entity is present. If the entity is not found in the 
persistence context, we execute the `SQL` select statement and put the retrieved entity in the persistence context.

That means that the `MovieServiceOne.findMovie()` found an empty persistence context, it executed the `SQL` select statement and populated the persistence context. Since the `MovieServiceTwo.findMovie()` is using the same `EntityManager`, it found the entity in the persistence context and
that's why the `MovieServiceTwo.findMovie()` method didn't execute any `SQL` statements.

### The `readOnly` attribute

We can mark the transaction as being read-only by specifying the `readOnly` attribute, like this: `@Transactional(readOnly = true)`. 
Read-only transactions are considered to be more performant and another effect of this attribute is that `Hibernate's` dirty-checking mechanisms would be disabled.
When fetching an entity, `Hibernate` apart from the fact that it maps the `ResultSet` to a Java object, it creates a snapshot of the `ResultSet` so that it can use it later, at flush-time to determine if the entity is dirty (meaning we need to update the entity).

With read-only transactions, this actions doesn't take place and this is precisely the reason of performance improvement. 
Let's check it. Take a look at the example below:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.findMovie("aa3e4567-e89b-12d3-b457-5267141750aa");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final EntityManager entityManager;

    public MovieService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional(readOnly = true)
    protected Movie findMovie(String id) {
        Movie movie = entityManager.find(Movie.class, id);
        log.debug("Found movie: {}", movie);
        movie.setName(movie.getName() + "!");
        return movie;
    }
}
```

In the example above, in a read-only transaction we fetch a `Movie` entity and change its name (making the entity dirty). If we had a regular "write" transaction, this will trigger an SQL update on the movie table. Let's check the logs to see what happened in our case: 

```
2022-05-13 19:22:55.713 DEBUG 1353141 --- [main] o.s.orm.jpa.JpaTransactionManager: Creating new transaction with name [inc.evil.spring.tx.management.MovieService.findMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,readOnly
2022-05-13 19:22:55.713 DEBUG 1353141 --- [main] o.s.orm.jpa.JpaTransactionManager: Opened new EntityManager [SessionImpl(196161345<open>)] for JPA transaction
2022-05-13 19:22:55.715 DEBUG 1353141 --- [main] o.s.orm.jpa.JpaTransactionManager: Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@75dd0f94]
Hibernate: 
    select
        movie0_.id as id1_1_0_,
        movie0_.name as name2_1_0_ 
    from
        movies movie0_ 
    where
        movie0_.id=?
2022-05-13 19:22:55.724 DEBUG 1353141 --- [main] i.e.s.tx.management.MovieService    : Found movie: Movie{name='Pulp Fiction!', id='aa3e4567-e89b-12d3-b457-5267141750aa'}
2022-05-13 19:22:55.725 DEBUG 1353141 --- [main] o.s.orm.jpa.JpaTransactionManager   : Initiating transaction commit
```

No SQL updates spotted, signifying that the dirty-checking mechanism was indeed disabled.

{{< admonition type=tip title="Good idea" open=true >}}
If the transaction only reads data, mark it as read-only. This wil not only improve the performance, but also serve as a form of documentation.
{{< /admonition >}}

Now, what will happen if in a `readOnly` transaction we'll try to persist an entity, as shown in the example below?

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.findMovie("aa3e4567-e89b-12d3-b457-5267141750aa");
    };
  }
}

@Service
@Slf4j
class MovieService {
    private final EntityManager entityManager;

    public MovieService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional(readOnly = true)
    protected Movie findMovie(String id) {
        Movie movie = entityManager.find(Movie.class, id);
        log.debug("Found movie: {}", movie);
        Movie newMovie = Movie.builder()
                .name("Joker")
                .build();
        entityManager.persist(newMovie);
        return movie;
    }
}
```

Well, the `EntityManager.persist()` will be silently ignored (without throwing any exceptions), see the logs below:

```
2022-05-13 19:35:50.030 DEBUG 1353975 --- [main] o.s.orm.jpa.JpaTransactionManager: Creating new transaction with name [inc.evil.spring.tx.management.MovieService.findMovie]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,readOnly
2022-05-13 19:35:50.030 DEBUG 1353975 --- [main] o.s.orm.jpa.JpaTransactionManager: Opened new EntityManager [SessionImpl(228806320<open>)] for JPA transaction
2022-05-13 19:35:50.032 DEBUG 1353975 --- [main] o.s.orm.jpa.JpaTransactionManager: Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@2dc21583]
Hibernate: 
    select
        movie0_.id as id1_1_0_,
        movie0_.name as name2_1_0_ 
    from
        movies movie0_ 
    where
        movie0_.id=?
2022-05-13 19:35:50.042 DEBUG 1353975 --- [main] i.e.s.tx.management.MovieService   : Found movie: Movie{name='Pulp Fiction!', id='aa3e4567-e89b-12d3-b457-5267141750aa'}
2022-05-13 19:35:50.047 DEBUG 1353975 --- [main] o.s.orm.jpa.JpaTransactionManager  : Initiating transaction commit
```

This behavior actually varies from one `PlatformTransactionManager` to another (there are implementations which do throw exceptions in this case), so don't depend on it. 
To be more precise, the `DataSourceTransactionManager` is the transaction manager which throws exceptions in this case. Let's try it out:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
  }

  @Bean
  public CommandLineRunner commandLineRunner(MovieService movieService) {
    return args -> {
      movieService.findMovie("aa3e4567-e89b-12d3-b457-5267141750aa");
    };
  }

  @Bean
  public PlatformTransactionManager dataSourceTransactionManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
  }
}

@Service
@Slf4j
class MovieService {
  private final JdbcTemplate jdbcTemplate;

  public MovieServiceOne(DataSource dataSource) {
    this.jdbcTemplate =  new JdbcTemplate(dataSource);
  }

  @Transactional(transactionManager = "dataSourceTransactionManager", readOnly = true)
  protected Movie findMovie(String id) {
    jdbcTemplate.update("insert into movies(id, name) values ('1', 'Joker')");
    return null;
  }
}
```

So we're trying to do a SQL insert in a read-only transaction. Also pay attention to the fact that we've specified a different transaction manager using the `transactionManager` attribute on the `@Transactional` annotation.

The exception we've got looks like this:

```
Caused by: org.postgresql.util.PSQLException: ERROR: cannot execute INSERT in a read-only transaction
	at org.postgresql.core.v3.QueryExecutorImpl.receiveErrorResponse(QueryExecutorImpl.java:2675) ~[postgresql-42.3.4.jar:42.3.4]
```

## Conclusion

In this blog post we made a gentle introduction on `Spring's` declarative and programmatic transaction management approaches, looked at all the propagation levels, what's the default commit and rollback behavior (and how to configure it), 
discussed that by default only `public` methods can be transactional, but with a bit of configuration
we can enable the transactional behavior for `package-private` and `protected` methods as well.

We also compared a bit the programmatic and declarative approaches and saw that they aren't 100% equivalent, and we also saw that read-only transactions disable the dirty-checking mechanism.

We'll dive deeper in another blog post where we'll look at some puzzlers and limitations of declarative transaction management.

---
title: "Introduction to declarative transaction management in Spring Framework"
date: 2022-05-10T08:54:47+03:00
draft: true
author: "Andrei Rosca"
tags: ["Spring Framework", "Transaction management"]
categories: ["Spring Framework"]
---
## Introduction

In this blog post we are going to explore the internals of Spring's declarative transaction management. We'll start with the basics, and then we'll dive deeper, looking at the internals and some potential pitfalls which
we can run into. But first, let's discuss a bit why do we even bother with transactions in the first place?

## Why do we need transactions?

The most common reason for using transactions in an application is to maintain a high degree of data integrity and consistency.
If we're unconcerned about the quality of our data, we needn't concern ourselves with transactions.

Transaction management is ubiquitous, it is present is every Java application which uses a database. The Spring Framework out of the box provides a lot of mechanisms to manage transactions and though 
it makes our lives easier, it is quite important to understand how it works and what happens under the hood since there are some pitfalls which can lead to undesired results.
Let's take a closer look at what transaction management mechanism Spring provides and how we can use them.

Suppose we have the following `JPA` entity:

```java
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

With the following `Spring Data JPA` repository:

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
way the `JpaTransactionManager` will publish logs every time it opens, commits or rolls-back transactions.
Setting the debug log level can be accomplished by adding the following property in the `application.properties` file:

```properties
#Logging properties
logging.level.org.springframework.orm.jpa=debug
```

If we run the application now, we can observe the following in the logs:

```
2022-05-10 18:23:56.122 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-10 18:23:56.122 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(1337659716<open>)] for JPA transaction
2022-05-10 18:23:56.125 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@2c34402]
2022-05-10 18:23:56.134 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction commit
2022-05-10 18:23:56.135 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Committing JPA transaction on EntityManager [SessionImpl(1337659716<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-10 18:23:56.144 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(1337659716<open>)] after transaction



2022-05-10 18:23:56.144 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-10 18:23:56.145 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(1862946352<open>)] for JPA transaction
2022-05-10 18:23:56.145 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@50ff7063]
2022-05-10 18:23:56.145 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction commit
2022-05-10 18:23:56.145 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Committing JPA transaction on EntityManager [SessionImpl(1862946352<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-10 18:23:56.146 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(1862946352<open>)] after transaction


2022-05-10 18:23:56.146 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-10 18:23:56.146 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(654299840<open>)] for JPA transaction
2022-05-10 18:23:56.147 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@d28c214]
2022-05-10 18:23:56.147 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction commit
2022-05-10 18:23:56.147 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Committing JPA transaction on EntityManager [SessionImpl(654299840<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-10 18:23:56.149 DEBUG 336867 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(654299840<open>)] after transaction

```

We can observe that the sequence: `Creating new transaction with name` and `Committing JPA transaction on EntityManager` is present `3` times.
That means that every movie is saved in a separate transaction and this also means that the `MovieService.saveMovies()` method is
not atomic. If for example run into some issue inserting the third movie (for example a unique constraint is violated), only the third transaction will be rolled-back, and 
we'll end up with 2 movies in the database. If we expected that the `MovieService.saveMovies()` is atomic, meaning it "inserts all movies or nothing", that's certainly not the case.

### How to fix it?

To make the `MovieService.saveMovies()` method atomic and obtain the "all or nothing" behavior, we can just annotate the method with `@Transactional` annotation. It will look like this:

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
2022-05-10 19:00:25.049 DEBUG 350995 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.saveMovies]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-10 19:00:25.049 DEBUG 350995 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(2139895366<open>)] for JPA transaction
2022-05-10 19:00:25.050 DEBUG 350995 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@376b5cb2]
2022-05-10 19:00:25.056 DEBUG 350995 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Found thread-bound EntityManager [SessionImpl(2139895366<open>)] for JPA transaction
2022-05-10 19:00:25.056 DEBUG 350995 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Participating in existing transaction
2022-05-10 19:00:25.064 DEBUG 350995 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Found thread-bound EntityManager [SessionImpl(2139895366<open>)] for JPA transaction
2022-05-10 19:00:25.064 DEBUG 350995 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Participating in existing transaction
2022-05-10 19:00:25.065 DEBUG 350995 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Found thread-bound EntityManager [SessionImpl(2139895366<open>)] for JPA transaction
2022-05-10 19:00:25.065 DEBUG 350995 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Participating in existing transaction
2022-05-10 19:00:25.065 DEBUG 350995 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction commit
2022-05-10 19:00:25.065 DEBUG 350995 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Committing JPA transaction on EntityManager [SessionImpl(2139895366<open>)]
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
2022-05-10 19:00:25.079 DEBUG 350995 --- [           main] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(2139895366<open>)] after transaction
```

Notice that the `Creating new transaction with name` sequence is present only once this time, meaning we have a single transaction, as we expected.

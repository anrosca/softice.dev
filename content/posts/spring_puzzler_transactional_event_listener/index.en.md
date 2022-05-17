---
title: "Spring puzzler: the @TransactionalEventListener"
date: 2022-05-16T23:01:00+03:00
draft: false
author: "Andrei Rosca"
tags: ["Spring Framework", "Declarative transaction management", "@TransactionalEventListener"]
categories: ["Spring Framework"]
---

## Introduction

Today we're going to take a look at a new `Spring` `@Transactional` puzzler involving the `@TransactionalEventListener`.
It's an old quirk of `Spring` related to transaction-bound events (both declarative and programmatic ones)
and though not commonly experienced, encountering it can leave you confused for hours. 
Let's have a look.

## The puzzler

Suppose we have the following code:

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
            Movie movie = Movie.builder()
                    .name("Joker")
                    .build();
            movieService.save(movie);
        };
    }
}

@Service
@Slf4j
class MovieService {
    private final MovieRepository movieRepository;

    public MovieService(MovieRepository movieRepository) {
        this.movieRepository = movieRepository;
    }

    @Transactional
    public Movie save(Movie movie) {
        Movie savedMovie = movieRepository.save(movie);
        log.debug("Saved movie: {}", savedMovie);
        return savedMovie;
    }
}
```

Nothing fancy so far. We have a `CommandLineRunner` which calls the `MovieService.save()` method which saves to the
database a `Movie` `JPA` entity using `Spring-Data-JPA`.

The `Movie` entity looks like this btw:

```java
@Entity
@Table(name = "movies")
public class Movie extends AbstractEntity {
    private String name;
    
    //Getters, setters, toString and the builder are omitted for brevity
```

Now let's say we want to implement some form of auditing. Every time a `Movie` entity is saved, we want to save also 
a `MovieAudit` entity, just for inspection purposes.

The `MovieAudit` entity looks like this:

```java
@Entity
@Table(name = "movie_audit")
public class MovieAudit extends AbstractEntity {
    private String name;
    private String movieId;

    public static MovieAudit from(Movie movie) {
        return MovieAudit.builder()
                .movieId(movie.getId())
                .name(movie.getName())
                .build();
    }

    //Getters, setters, toString and the builder are omitted for brevity
```

Now in order to implement the required behavior, we can inject the repository for the `MovieAudit` entity directly into
our `MovieService` and save our `MovieAudit` along with the `Movie` entity in the same transaction, like this: 

```java
@Service
@Slf4j
class MovieService {
    private final MovieRepository movieRepository;
    private final MovieAuditRepository movieAuditRepository;

    public MovieService(MovieRepository movieRepository, MovieAuditRepository movieAuditRepository) {
        this.movieRepository = movieRepository;
        this.movieAuditRepository = movieAuditRepository;
    }

    @Transactional
    public Movie save(Movie movie) {
        Movie savedMovie = movieRepository.save(movie);
        log.debug("Saved movie: {}", savedMovie);
        movieAuditRepository.save(MovieAudit.from(savedMovie));
        return savedMovie;
    }
}
```

This approach doesn't look that good since our `MovieService` looks a bit messier. Let's try to move `MovieAudit` logic
our of the `MovieService`. But how? `Spring events` to the rescue!

## Using `Spring events`

Instead of saving the `MovieAudit` from the `MovieService`, we can adjust the `MovieService` to publish a Spring event. Just like this:

```java
@Service
@Slf4j
class MovieService {
    private final MovieRepository movieRepository;
    private final ApplicationEventPublisher eventPublisher;

    public MovieService(MovieRepository movieRepository, ApplicationEventPublisher eventPublisher) {
        this.movieRepository = movieRepository;
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public Movie save(Movie movie) {
        Movie savedMovie = movieRepository.save(movie);
        log.debug("Saved movie: {}", savedMovie);
        eventPublisher.publishEvent(new MovieSavedEvent(savedMovie));
        return savedMovie;
    }
}
```

On one hand, our `MovieService` didn't become simpler but this approach is more flexible since we can plug-in as many
event listeners as we want. For example if in the future we'll be required to do some other action every time a `Movie` entity
is saved, we can just add a new event-listener.

Our event-listener which persists the `MovieAudit` can look like this:

```java
@Slf4j
@Component
class MovieAuditEventListener {
    private final MovieAuditRepository movieAuditRepository;

    public MovieAuditEventListener(MovieAuditRepository movieAuditRepository) {
        this.movieAuditRepository = movieAuditRepository;
    }

    @EventListener(MovieSavedEvent.class)
    @Transactional
    public void on(MovieSavedEvent event) {
        log.debug("Received event: {}", event);
        MovieAudit movieAudit = MovieAudit.from(event.getMovie());
        movieAuditRepository.save(movieAudit);
        log.debug("Saved movie audit: {}", movieAudit);
    }
}
```

Let's run the example and check the logs:

```
2022-05-16 23:51:14.100 DEBUG 1641040 --- [main] o.s.orm.jpa.JpaTransactionManager    : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.save]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-16 23:51:14.100 DEBUG 1641040 --- [main] o.s.orm.jpa.JpaTransactionManager    : Opened new EntityManager [SessionImpl(1483155688<open>)] for JPA transaction
2022-05-16 23:51:14.102 DEBUG 1641040 --- [main] o.s.orm.jpa.JpaTransactionManager    : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@f4f843f]
2022-05-16 23:51:14.106 DEBUG 1641040 --- [main] o.s.orm.jpa.JpaTransactionManager    : Found thread-bound EntityManager [SessionImpl(1483155688<open>)] for JPA transaction
2022-05-16 23:51:14.106 DEBUG 1641040 --- [main] o.s.orm.jpa.JpaTransactionManager    : Participating in existing transaction
2022-05-16 23:51:14.114 DEBUG 1641040 --- [main] i.e.spring.tx.management.MovieService: Saved movie: Movie{name='Joker', id='e4d96c77-fbe4-4407-af35-e11176a79da5'}
2022-05-16 23:51:14.115 DEBUG 1641040 --- [main] o.s.orm.jpa.JpaTransactionManager    : Found thread-bound EntityManager [SessionImpl(1483155688<open>)] for JPA transaction
2022-05-16 23:51:14.115 DEBUG 1641040 --- [main] o.s.orm.jpa.JpaTransactionManager    : Participating in existing transaction
2022-05-16 23:51:14.117 DEBUG 1641040 --- [main] i.e.s.t.m.MovieAuditEventListener    : Received event: MovieSavedEvent{movie=Movie{name='Joker', id='e4d96c77-fbe4-4407-af35-e11176a79da5'}}
2022-05-16 23:51:14.117 DEBUG 1641040 --- [main] o.s.orm.jpa.JpaTransactionManager    : Found thread-bound EntityManager [SessionImpl(1483155688<open>)] for JPA transaction
2022-05-16 23:51:14.117 DEBUG 1641040 --- [main] o.s.orm.jpa.JpaTransactionManager    : Participating in existing transaction
2022-05-16 23:51:14.118 DEBUG 1641040 --- [main] i.e.s.t.m.MovieAuditEventListener    : Saved movie audit: MovieAudit{name='Joker', id='b73e3f40-7cad-4821-91ca-c867bd1e79da', movieId='e4d96c77-fbe4-4407-af35-e11176a79da5'}
2022-05-16 23:51:14.119 DEBUG 1641040 --- [main] o.s.orm.jpa.JpaTransactionManager    : Initiating transaction commit
2022-05-16 23:51:14.119 DEBUG 1641040 --- [main] o.s.orm.jpa.JpaTransactionManager    : Committing JPA transaction on EntityManager [SessionImpl(1483155688<open>)]
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
        movie_audit
        (movie_id, name, id) 
    values
        (?, ?, ?)
2022-05-16 23:51:14.130 DEBUG 1641040 --- [main] o.s.orm.jpa.JpaTransactionManager    : Closing JPA EntityManager [SessionImpl(1483155688<open>)] after transaction
```

As we can see, everything works as expected and both the `Movie` and `MovieAudit` were persisted to the database.

One important thing to note here is that `Spring events` by default are synchronous. When the `ApplicationEventPublisher.publishEvent()`
method is called, all of the event-listeners for the published event are synchronously called. Since the `MovieAuditEventListener` is
synchronously invoked and its `@EventListener` method is also annotated with `@Transactional` (with the default propagation which is `Propagation.REQUIRED`), 
this means that our `MovieAuditEventListener` will join the transaction created by the `MovieService`.

But what if we don't want the `MovieAudit` to be saved in the same transaction as the `Movie` entity? The rationale could be that
since the event-listener joins an existing transaction, we can't be certain what "fate" that transaction will have. It can be rolled-back.

Also maybe our `MovieService` is a mission-critical service and we want it's transactions as short-lived as possible. Adding `MovieAudit` logic 
to it can have a noticeable impact. And thus, we decide to use a `@TransactionalEventListener`.

## The `@TransactionalEventListener` annotation

A `@TransactionalEventListener` is a similar to a regular `@EventListener`, the difference being that the events are transaction-bounded.
This means that the event listeners in this case are not immediately-invoked, but are called when the transaction from which they were thrown
changes its phase. Here are the possible phases:

- `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`: the default. The listeners are invoked after the transaction commit has completed successfully.
- `@TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)`:the listeners are invoked after the transaction has been rolled-back.
- `@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)`: the listeners are invoked before the transaction will be committed.
- `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)`: the listeners are invoked after the transaction will be completed, so both after commit and rollback.

In our example we'll use `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)` since we want our `MovieAudit` entity saved
after the transaction of the `MovieService.save()` method will be committed.

Our listener will look something like this:

```java
@Slf4j
@Component
class MovieAuditEventListener {
    private final MovieAuditRepository movieAuditRepository;

    public MovieAuditEventListener(MovieAuditRepository movieAuditRepository) {
        this.movieAuditRepository = movieAuditRepository;
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional
    public void on(MovieSavedEvent event) {
        log.debug("Received event: {}", event);
        MovieAudit movieAudit = MovieAudit.from(event.getMovie());
        movieAuditRepository.save(movieAudit);
        log.debug("Saved movie audit: {}", movieAudit);
    }
}
```

Now let's check the logs:

```
2022-05-16 23:53:07.427 DEBUG 1641223 --- [main] o.s.orm.jpa.JpaTransactionManager    : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.save]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-16 23:53:07.427 DEBUG 1641223 --- [main] o.s.orm.jpa.JpaTransactionManager    : Opened new EntityManager [SessionImpl(1305564302<open>)] for JPA transaction
2022-05-16 23:53:07.428 DEBUG 1641223 --- [main] o.s.orm.jpa.JpaTransactionManager    : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@3b362f1]
2022-05-16 23:53:07.432 DEBUG 1641223 --- [main] o.s.orm.jpa.JpaTransactionManager    : Found thread-bound EntityManager [SessionImpl(1305564302<open>)] for JPA transaction
2022-05-16 23:53:07.432 DEBUG 1641223 --- [main] o.s.orm.jpa.JpaTransactionManager    : Participating in existing transaction
2022-05-16 23:53:07.442 DEBUG 1641223 --- [main] i.e.spring.tx.management.MovieService: Saved movie: Movie{name='Joker', id='dc766b33-2ba7-4989-bea2-99734243f0c9'}
2022-05-16 23:53:07.443 DEBUG 1641223 --- [main] o.s.orm.jpa.JpaTransactionManager    : Initiating transaction commit
2022-05-16 23:53:07.443 DEBUG 1641223 --- [main] o.s.orm.jpa.JpaTransactionManager    : Committing JPA transaction on EntityManager [SessionImpl(1305564302<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-16 23:53:07.454 DEBUG 1641223 --- [main] o.s.orm.jpa.JpaTransactionManager    : Found thread-bound EntityManager [SessionImpl(1305564302<open>)] for JPA transaction
2022-05-16 23:53:07.454 DEBUG 1641223 --- [main] o.s.orm.jpa.JpaTransactionManager    : Participating in existing transaction
2022-05-16 23:53:07.456 DEBUG 1641223 --- [main] i.e.s.t.m.MovieAuditEventListener    : Received event: MovieSavedEvent{movie=Movie{name='Joker', id='dc766b33-2ba7-4989-bea2-99734243f0c9'}}
2022-05-16 23:53:07.457 DEBUG 1641223 --- [main] o.s.orm.jpa.JpaTransactionManager    : Found thread-bound EntityManager [SessionImpl(1305564302<open>)] for JPA transaction
2022-05-16 23:53:07.457 DEBUG 1641223 --- [main] o.s.orm.jpa.JpaTransactionManager    : Participating in existing transaction
2022-05-16 23:53:07.458 DEBUG 1641223 --- [main] i.e.s.t.m.MovieAuditEventListener    : Saved movie audit: MovieAudit{name='Joker', id='db2c13dc-7839-4cce-965d-20c4286cb786', movieId='dc766b33-2ba7-4989-bea2-99734243f0c9'}
2022-05-16 23:53:07.459 DEBUG 1641223 --- [main] o.s.orm.jpa.JpaTransactionManager    : Closing JPA EntityManager [SessionImpl(1305564302<open>)] after transaction
```

{{< figure src="aaand_its_gone.jpeg" alt="aaand_its_gone.jpeg" >}}

The `SQL` insert for the `MovieAudit` entity magically disappeared. From the logs we see a single `SQL` insert statement into the `movies` table and
no traces of the `SQL` insert into the `movie_audit` table. 

So what actually happened?

## Explanation

The short version is that the `MovieService` has created a transaction and has bound it to the current thread. Then the
`MovieService's` proxy commits the transaction and invokes the `@TransactionalEventListener`. The thing is that at this point,
the initial transaction is already committed but it wasn't yet cleaned-up (or unbound from the current thread). Because of that,
when the `MovieAuditEventListener` is called, it thinks that there's an existing transaction and it tries to join it. The listener thinks that there's
an existing transaction because it can find the thread-local data of the previously committed transaction.

This way `Spring` is fooled and it joins a committed transaction. Now, the `SQL` insert for the `MovieAudit` was lost since `Hibernate` usually executes inserts
at flush-time (which happens at the transaction commit), but since the transaction was committed a long time ago, nothing happens. We can't commit a transaction twice.

{{< admonition type=note title="In-depth explanation" open=false >}}

The `TransactionSynchronizationManager` class is the place where all of the thread-bounded transaction data is stored, like this:

```java
package org.springframework.transaction.support;

public abstract class TransactionSynchronizationManager {

	private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<>("Transactional resources");

	private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
			new NamedThreadLocal<>("Transaction synchronizations");

	private static final ThreadLocal<String> currentTransactionName =
			new NamedThreadLocal<>("Current transaction name");

	private static final ThreadLocal<Boolean> currentTransactionReadOnly =
			new NamedThreadLocal<>("Current transaction read-only status");

	private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
			new NamedThreadLocal<>("Current transaction isolation level");

	private static final ThreadLocal<Boolean> actualTransactionActive =
			new NamedThreadLocal<>("Actual transaction active");
    //...
```

When the `PlatformTransactionManager` (in our case it's the `JpaTransactionManager`) needs to see if there's an active transaction, it checks the `TransactionSynchronizationManager` to see if we have any thread-bounded resources, like this:

```java
package org.springframework.orm.jpa;

public class JpaTransactionManager extends AbstractPlatformTransactionManager
        implements ResourceTransactionManager, BeanFactoryAware, InitializingBean {
    //...
    @Override
    protected Object doGetTransaction() {
        JpaTransactionObject txObject = new JpaTransactionObject();
        txObject.setSavepointAllowed(isNestedTransactionAllowed());

        EntityManagerHolder emHolder = (EntityManagerHolder)
                TransactionSynchronizationManager.getResource(obtainEntityManagerFactory()); //Do we have already an EntityManger?
        if (emHolder != null) {
            if (logger.isDebugEnabled()) {
                logger.debug("Found thread-bound EntityManager [" + emHolder.getEntityManager() +
                        "] for JPA transaction");
            }
            txObject.setEntityManagerHolder(emHolder, false); //If found an existing EntityManager, set it on the new transaction object
        }
        if (getDataSource() != null) {
            ConnectionHolder conHolder = (ConnectionHolder)
                    TransactionSynchronizationManager.getResource(getDataSource()); //Find the JDBC Connection holder
            txObject.setConnectionHolder(conHolder); //Set it on the new transaction object
        }
        return txObject;
    }
```

At this point, nothing fancy happened. We've just collected the data about the existing transaction (if any). `Spring` has instantiated a `JpaTransactionObject` instance
and has populated it with the thread-bound `EntityManagerHolder` (which has an `EntityManager`) and a `ConnectionHolder` (which has a JDBC `Connection`). But these 2 values can have the value
of `null` if there isn't an existing transaction.

Now `Spring` checks if we do have an active transaction, like this:

```java
package org.springframework.transaction.support;

public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {
    //...
    @Override
	public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
			throws TransactionException {
        //...
		Object transaction = doGetTransaction(); // We saw this method in the previous code snippet
        //...
		if (isExistingTransaction(transaction)) {
			// Existing transaction found -> check propagation behavior to find out how to behave.
			return handleExistingTransaction(def, transaction, debugEnabled);
		}
        //...
```

`Spring` thinks that there's an existing transaction if the `JpaTransactionManager.isExistingTransaction()` tells so, like this:

```java
package org.springframework.orm.jpa;

public class JpaTransactionManager extends AbstractPlatformTransactionManager
        implements ResourceTransactionManager, BeanFactoryAware, InitializingBean {
    //...
	@Override
	protected boolean isExistingTransaction(Object transaction) {
		return ((JpaTransactionObject) transaction).hasTransaction();
	}
    //...
```

And the `JpaTransactionObject.hasTransaction()` method consults the `EntityManagerHolder` class, which we've obtained via the `TransactionSynchronizationManager`

```java
	private class JpaTransactionObject extends JdbcTransactionObjectSupport {
		public boolean hasTransaction() {
			return (this.entityManagerHolder != null && this.entityManagerHolder.isTransactionActive());
		}
    //...
```

And finally the `EntityManagerHolder` just checks a `boolean` flag:

```java
public class EntityManagerHolder extends ResourceHolderSupport {
	@Nullable
	private final EntityManager entityManager;
	private boolean transactionActive;
    
    //...

	protected boolean isTransactionActive() {
		return this.transactionActive;   //It is considered that we have an active transaction if this boolean flag is true
	}
    //...
```

So, `Spring` thinks that we have an active transaction if the `EntityManagerHolder` we've obtained from the `TransactionSynchronizationManager` (which was populated when a transaction was started)
has the `transactionActive` boolean field set to `true`.

When we call a `@Transactional` method and we don't have an active transaction already, the `TransactionSynchronizationManager` will be empty, so we won't get an `EntityManagerHolder` from it.
In this case `Spring` will create a new transaction. If the `TransactionSynchronizationManager` returns an `EntityManagerHolder` and is has the `transactionActive` flag set to `true`,
`Spring` thinks we have an active transaction and joins it.

Now let's see how the `@TransactionalEventListener` are invoked:

```java
package org.springframework.transaction.support;

public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {
    //...
    private void processCommit(DefaultTransactionStatus status) throws TransactionException {
        //...
        try {
                prepareForCommit(status);
                triggerBeforeCommit(status); // Trigger the @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)

                if (status.hasSavepoint()) {
                    //...
                }
                else if (status.isNewTransaction()) {
                    //...
                    doCommit(status); //Execute the actual commit
                }
               //...
            try {
                triggerAfterCommit(status); // Trigger the @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT) 
            }
            finally {
                triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED); // Trigger the @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION) 
            }
        }
        finally {
            cleanupAfterCompletion(status); // Clean up the TransactionSynchronizationManager. Here the EntityManagerHolder will be deleted from thread-local storage
        }
    }
    //...

    private void cleanupAfterCompletion(DefaultTransactionStatus status) {
        status.setCompleted();
        if (status.isNewSynchronization()) {
            TransactionSynchronizationManager.clear();
        }
        if (status.isNewTransaction()) {
            doCleanupAfterCompletion(status.getTransaction());
        }
        //...
    }
```

Did you notice that the `triggerAfterCommit(status)` method call which calls the `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)` happens before the `cleanupAfterCompletion(status)`?
Well, that's the explanation. Since the `TransactionSynchronizationManager` was not cleared and it still has the `EntityManagerHolder`
at the point when the `@TransactionalEventListener` are called, if the event listener wants a transaction it will think that there's an active transaction! And if the
`@TransactionalEventListener` will use the `Propagation.REQUIRED`, it will join the already-committed transaction and thus - all of it's work related to the database will be lost since
there won't be a flush of the persistence context, since the flush happens at commit-time but the commit was already executed. Effectively our listener has joined a dead transaction.

{{< /admonition >}}

## How to fix it?

In order to fix it, we'll need to change the propagation level for the `MovieAuditEventListener` and use
`propagation = Propagation.REQUIRES_NEW` so that we force it to create a new transaction, like this:

```java
@Slf4j
@Component
class MovieAuditEventListener {
    private final MovieAuditRepository movieAuditRepository;

    public MovieAuditEventListener(MovieAuditRepository movieAuditRepository) {
        this.movieAuditRepository = movieAuditRepository;
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void on(MovieSavedEvent event) {
        log.debug("Received event: {}", event);
        MovieAudit movieAudit = MovieAudit.from(event.getMovie());
        movieAuditRepository.save(movieAudit);
        log.debug("Saved movie audit: {}", movieAudit);
    }
}
```

Let's run it and check the logs to see if it actually works:

```
2022-05-17 09:05:21.798 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.save]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-17 09:05:21.799 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Opened new EntityManager [SessionImpl(1187406578<open>)] for JPA transaction
2022-05-17 09:05:21.800 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@18b40fe6]
2022-05-17 09:05:21.804 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Found thread-bound EntityManager [SessionImpl(1187406578<open>)] for JPA transaction
2022-05-17 09:05:21.804 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Participating in existing transaction
2022-05-17 09:05:21.812 DEBUG 1666084 --- [main] i.e.spring.tx.management.MovieService: Saved movie: Movie{name='Joker', id='c2322370-471e-4512-96a5-d04941ecff4f'}
2022-05-17 09:05:21.813 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Initiating transaction commit
2022-05-17 09:05:21.813 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Committing JPA transaction on EntityManager [SessionImpl(1187406578<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-17 09:05:21.824 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Found thread-bound EntityManager [SessionImpl(1187406578<open>)] for JPA transaction
2022-05-17 09:05:21.824 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Suspending current transaction, creating new transaction with name [inc.evil.spring.tx.management.MovieAuditEventListener.on]
2022-05-17 09:05:21.824 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Opened new EntityManager [SessionImpl(2117195067<open>)] for JPA transaction
2022-05-17 09:05:21.825 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@68d8eb4f]
2022-05-17 09:05:21.828 DEBUG 1666084 --- [main] i.e.s.t.m.MovieAuditEventListener    : Received event: MovieSavedEvent{movie=Movie{name='Joker', id='c2322370-471e-4512-96a5-d04941ecff4f'}}
2022-05-17 09:05:21.828 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Found thread-bound EntityManager [SessionImpl(2117195067<open>)] for JPA transaction
2022-05-17 09:05:21.828 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Participating in existing transaction
2022-05-17 09:05:21.829 DEBUG 1666084 --- [main] i.e.s.t.m.MovieAuditEventListener    : Saved movie audit: MovieAudit{name='Joker', id='5125bd9f-e854-496d-822f-d3a441c02f22', movieId='c2322370-471e-4512-96a5-d04941ecff4f'}
2022-05-17 09:05:21.830 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Initiating transaction commit
2022-05-17 09:05:21.830 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Committing JPA transaction on EntityManager [SessionImpl(2117195067<open>)]
Hibernate: 
    insert 
    into
        movie_audit
        (movie_id, name, id) 
    values
        (?, ?, ?)
2022-05-17 09:05:21.832 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Closing JPA EntityManager [SessionImpl(2117195067<open>)] after transaction
2022-05-17 09:05:21.832 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Resuming suspended transaction after completion of inner transaction
2022-05-17 09:05:21.832 DEBUG 1666084 --- [main] o.s.orm.jpa.JpaTransactionManager    : Closing JPA EntityManager [SessionImpl(1187406578<open>)] after transaction
```

And indeed it works! We can clearly see that we have 2 `SQL` insert statements now - one for the `Movie` entity and
another one for the `MovieAudit`, as expected.

{{< admonition type=note title="Note" open=false >}}

Instead of using the declarative approach (using the `@TransactionalEventListener`) to register the transaction-bound event listeners, we can register our listeners
programmatically, using the `TransactionSynchronizationManager.registerSynchronization()` method. Will it help solve the issue?

Well, switching to the programmatic transaction-bound events will not help. We'll get exactly the same behavior, see below:

```java
@Service
@Slf4j
class MovieService {
    private final MovieRepository movieRepository;
    private final MovieAuditRepository movieAuditRepository;

    public MovieService(MovieRepository movieRepository, MovieAuditRepository movieAuditRepository) {
        this.movieRepository = movieRepository;
        this.movieAuditRepository = movieAuditRepository;
    }

    @Transactional
    public Movie save(Movie movie) {
        Movie savedMovie = movieRepository.save(movie);
        log.debug("Saved movie: {}", savedMovie);
        TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
            @Override
            public void afterCommit() {
                MovieAudit movieAudit = MovieAudit.from(savedMovie);
                movieAuditRepository.save(movieAudit);
                log.debug("Saved movie audit: {}", movieAudit);
            }
        });
        return savedMovie;
    }
}
```

We can check the logs and observe that the `SQL` insert for the `MovieAudit` is still lost:

```
022-05-17 13:02:05.208 DEBUG 1698985 ---  [main] o.s.orm.jpa.JpaTransactionManager    : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.save]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-17 13:02:05.208 DEBUG 1698985 --- [main] o.s.orm.jpa.JpaTransactionManager    : Opened new EntityManager [SessionImpl(926544841<open>)] for JPA transaction
2022-05-17 13:02:05.212 DEBUG 1698985 --- [main] o.s.orm.jpa.JpaTransactionManager    : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@249b54af]
2022-05-17 13:02:05.222 DEBUG 1698985 --- [main] o.s.orm.jpa.JpaTransactionManager    : Found thread-bound EntityManager [SessionImpl(926544841<open>)] for JPA transaction
2022-05-17 13:02:05.222 DEBUG 1698985 --- [main] o.s.orm.jpa.JpaTransactionManager    : Participating in existing transaction
2022-05-17 13:02:05.240 DEBUG 1698985 --- [main] i.e.spring.tx.management.MovieService: Saved movie: Movie{name='Joker', id='6258154d-2548-470e-b560-0fb087f4fb4d'}
2022-05-17 13:02:05.241 DEBUG 1698985 --- [main] o.s.orm.jpa.JpaTransactionManager    : Initiating transaction commit
2022-05-17 13:02:05.242 DEBUG 1698985 --- [main] o.s.orm.jpa.JpaTransactionManager    : Committing JPA transaction on EntityManager [SessionImpl(926544841<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-17 13:02:05.261 DEBUG 1698985 --- [main] o.s.orm.jpa.JpaTransactionManager    : Found thread-bound EntityManager [SessionImpl(926544841<open>)] for JPA transaction
2022-05-17 13:02:05.261 DEBUG 1698985 --- [main] o.s.orm.jpa.JpaTransactionManager    : Participating in existing transaction
2022-05-17 13:02:05.262 DEBUG 1698985 --- [main] i.e.spring.tx.management.MovieService: Saved movie audit: MovieAudit{name='Joker', id='973ba59f-9ea6-44db-b24f-17be6ed2faba', movieId='6258154d-2548-470e-b560-0fb087f4fb4d'}
2022-05-17 13:02:05.263 DEBUG 1698985 --- [main] o.s.orm.jpa.JpaTransactionManager    : Closing JPA EntityManager [SessionImpl(926544841<open>)] after transaction
```

{{< /admonition >}}


## Conclusion

In this blog post we've looked at an odd quirk of `Spring` regarding the `@TransactionalEventListener` and we saw that
these listeners have to use `@Transactional(propagation = Propagation.REQUIRES_NEW)` if they want to use transactions, otherwise it won't work and they will join a
dead, committed transaction and this will make them lose all of their work related to the database.

We also saw that the programmatic approach - using the `TransactionSynchronizationManager.registerSynchronization()` does not help.

See you later, in another blog post.

The code is available on [GitHub](https://github.com/anrosca/spring_puzzler_transactional_event_listener).

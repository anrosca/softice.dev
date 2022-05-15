---
title: "Spring puzzler: transactional @PostConstruct methods"
date: 2022-05-15T16:40:47+03:00
draft: false
author: "Andrei Rosca"
tags: ["Spring Framework", "Declarative transaction management", "Programmatic transaction management"]
categories: ["Spring Framework"]
---

## Introduction

Today we'll be looking at a `Spring` puzzler - transactional `@PostContruct` methods. Though it's not a commonly used thing,
it can be useful to know some limitations of the Spring's declarative transaction management approach. 

## `@PostConstruct` methods

The `@PostConstruct` are called automatically by `Spring` after all of the bean's dependencies were injected. Let's look
at an example:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
    }
}

@Service
@Slf4j
class MovieService {
    @Autowired
    private EntityManager entityManager;

    public MovieService() {
        log.debug("entityManager: {}", entityManager);
    }

    @PostConstruct
    public void init() {
        log.debug("entityManager: {}", entityManager);
    }
}
```

We have a `MovieService` which is a `Spring` bean, and it used field-injection to get a dependency - the `EntityManager`. The question is,
when is our spring bean fully-initialized and ready to be used? 

Usually we consider that after calling the constructor, the instantiated object is in the right state so it can be safely used. Let's look
at the logs to see if that's the case:

```
2022-05-15 16:49:36.432 DEBUG 1451865 --- [main] i.e.spring.tx.management.MovieService: entityManager: null
2022-05-15 16:49:36.447 DEBUG 1451865 --- [main] i.e.spring.tx.management.MovieService: entityManager: Shared EntityManager proxy for target factory [org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean@79ab97fd]
```

As we can see from the logs, when the `MovieService` constructor is executing, the `entityManager` field still has the value of `null`. The `@PostConstruct` methods 
come to the rescue. They're invoked after all of the bean's dependencies were set, no matter the injection type used: constructor, field or setter.
Our logs prove that that's the case, the field `entityManager` is no longer `null` when the `@PostConstruct` method was called.

{{< admonition type=tip title="Tip" open=true >}}
It's considered a best practice to use constructor injection and steer clear of field injection since field-injection makes unit-testing way harder 
than it needs to be and it also prevents us from having immutable beans.
{{< /admonition >}}

## The puzzler

What will happen if we'll slightly modify our previous example and try to persist a `JPA` entity? Here are the options:

- [ ] The movie entity will be successfully persisted to the database
- [ ] The `init` method won't be called
- [ ] `BeanCreationException` will be thrown
- [ ] `TransactionRequiredException` will be thrown

Take a wild guess :)

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
    }
}

@Service
@Slf4j
class MovieService {
    @Autowired
    private EntityManager entityManager;

    public MovieService() {
        log.debug("entityManager: {}", entityManager);
    }

    @PostConstruct
    @Transactional
    public void init() {
        log.debug("entityManager: {}", entityManager);
        Movie movie = Movie.builder()
                .name("Joker")
                .build();
        entityManager.persist(movie);
    }
}
```

{{< admonition type=question title="Answer" open=false >}}
The right answer is:
- [ ] The movie entity will be successfully persisted to the database
- [ ] The `init` method won't be called
- [x] `BeanCreationException` will be thrown
- [ ] `TransactionRequiredException` will be thrown

Actually an exception will be thrown, specifically `BeanCreationException` with a cause of `TransactionRequiredException`. The `BeanCreationException` exception
is thrown when a `@PostConstruct` method throws an exception.

Here are the logs:

```
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'movieService': Invocation of init method failed; nested exception is javax.persistence.TransactionRequiredException: No EntityManager with actual transaction available for current thread - cannot reliably process 'persist' call
	at org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor.postProcessBeforeInitialization(InitDestroyAnnotationBeanPostProcessor.java:160) ~[spring-beans-5.3.19.jar:5.3.19]
```

It's worth stating that we deliberately used directly the `EntityManager`, since it doesn't create any transactions but its `persist`
method expects to be called with an active transaction. Using `Spring-data-jpa` here will help, since `Spring-data-jpa` creates
transactions as a last-resort, so using `Spring-data-jpa` will fix the puzzler.

{{< /admonition >}}

### Explanation

But what really happened? In one of our previous blog [posts](https://softice.dev/posts/introduction_to_declarative_tx_management/) we mentioned that Spring's declarative transaction management approach (using the `@Transactional` annotation) 
is based-on proxies by default. Here's a little refresher on how a proxy looks like:

{{< figure src="proxy_2.png" alt="The proxy pattern" >}}

Well, it turns out that at the time when the `@PostConstruct` method is invoked, the proxy for our `MovieService` was not created yet, so we can't
use the `@Transactional` annotation since there's no proxy to intercept the `init` method call and create a transaction for us. Very unfortunate, isn't it?

## How to fix it?

There are a couple of ways to fix this problem, let's explore the one by one.

### Using programmatic transaction management

Well, if during the `@PostConstruct` method call the proxy is not ready, one option would be to get rid of declarative transaction
management and use the programmatic one, since it doesn't rely on proxies, like this:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
    }
}

@Service
@Slf4j
class MovieService {
    @Autowired
    private EntityManager entityManager;

    private final PlatformTransactionManager transactionManager;

    public MovieService(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
        log.debug("entityManager: {}", entityManager);
    }

    @PostConstruct
    public void init() {
        TransactionTemplate transactionTemplate = new TransactionTemplate(transactionManager);
        transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus status) {
                log.debug("entityManager: {}", entityManager);
                Movie movie = Movie.builder()
                        .name("Joker")
                        .build();
                entityManager.persist(movie);
            }
        });
    }
}
```

This approach is certainly more verbose, but it allows us to fix the issue. Let's check the logs:

```
2022-05-16 07:02:29.688 DEBUG 1494469 --- [main] o.s.orm.jpa.JpaTransactionManager    : Creating new transaction with name [null]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-16 07:02:29.702 DEBUG 1494469 --- [main] o.s.orm.jpa.JpaTransactionManager    : Opened new EntityManager [SessionImpl(541713794<open>)] for JPA transaction
2022-05-16 07:02:29.704 DEBUG 1494469 --- [main] o.s.orm.jpa.JpaTransactionManager    : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@45b7c97f]
2022-05-16 07:02:29.704 DEBUG 1494469 --- [main] i.e.spring.tx.management.MovieService: entityManager: Shared EntityManager proxy for target factory [org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean@74123110]
2022-05-16 07:02:29.715 DEBUG 1494469 --- [main] o.s.orm.jpa.JpaTransactionManager    : Initiating transaction commit
2022-05-16 07:02:29.715 DEBUG 1494469 --- [main] o.s.orm.jpa.JpaTransactionManager    : Committing JPA transaction on EntityManager [SessionImpl(541713794<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-16 07:02:29.728 DEBUG 1494469 --- [main] o.s.orm.jpa.JpaTransactionManager    : Closing JPA EntityManager [SessionImpl(541713794<open>)] after transaction
```

As we can see, we do have a transaction and the `JPA` entity was successfully inserted.

### Listening to the `ContextRefreshedEvent` event

Another option would be to not use the `@PostConstruct` annotation, but listening to the `ContextRefreshedEvent` event, which 
is a spring event which is published after the Spring's `ApplicationContext` is refreshed. At this stage, the proxies for our
spring beans are guaranteed to be ready. It looks something like this:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
    }
}

@Service
@Slf4j
class MovieService {
    @Autowired
    private EntityManager entityManager;

    public MovieService() {
        log.debug("entityManager: {}", entityManager);
    }

    @EventListener(ContextRefreshedEvent.class)
    @Transactional
    public void init() {
        log.debug("entityManager: {}", entityManager);
        Movie movie = Movie.builder()
                .name("Joker")
                .build();
        entityManager.persist(movie);
    }
}
```

In the example above, instead of the `@PostConstruct` annotation, we're using the `@EventListener(ContextRefreshedEvent.class)` annotation.
Let's check the logs to see if our `JPA` entity was inserted properly:

```
2022-05-16 07:13:27.686 DEBUG 1495276 --- [main] o.s.orm.jpa.JpaTransactionManager    : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.init]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-16 07:13:27.687 DEBUG 1495276 --- [main] o.s.orm.jpa.JpaTransactionManager    : Opened new EntityManager [SessionImpl(235386075<open>)] for JPA transaction
2022-05-16 07:13:27.688 DEBUG 1495276 --- [main] o.s.orm.jpa.JpaTransactionManager    : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@6f2d3391]
2022-05-16 07:13:27.693 DEBUG 1495276 --- [main] i.e.spring.tx.management.MovieService: entityManager: Shared EntityManager proxy for target factory [org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean@69d103f0]
2022-05-16 07:13:27.698 DEBUG 1495276 --- [main] o.s.orm.jpa.JpaTransactionManager    : Initiating transaction commit
2022-05-16 07:13:27.698 DEBUG 1495276 --- [main] o.s.orm.jpa.JpaTransactionManager    : Committing JPA transaction on EntityManager [SessionImpl(235386075<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-16 07:13:27.705 DEBUG 1495276 --- [main] o.s.orm.jpa.JpaTransactionManager    : Closing JPA EntityManager [SessionImpl(235386075<open>)] after transaction
```

As we can see, we do have a transaction this time and the `JPA` entity was successfully inserted.

### Proxy self-injection

This is the messiest and odd-looking solution that actually works. Let's have a look:

```java
@Slf4j
@SpringBootApplication
public class SpringDeclarativeTxManagementApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
    }
}

@Service
@Slf4j
class MovieService {
    @Autowired
    private EntityManager entityManager;

    @Autowired
    private MovieService proxy;

    public MovieService() {
        log.debug("entityManager: {}", entityManager);
    }

    @PostConstruct
    public void init() {
        log.debug("entityManager: {}", entityManager);
        proxy.doInit();
    }

    @Transactional
    public void doInit() {
        Movie movie = Movie.builder()
                .name("Joker")
                .build();
        entityManager.persist(movie);
    }
}
```

In the example above, the `MovieService` tries to `@Autowire` itself. There are versions of the Spring framework (below `4.3`) in which this trick doesn't work.
Starting with Spring Framework `4.3`, support for self-injection with the `@Autowired` annotation was added, see [release notes here](https://spring.io/blog/2016/04/06/spring-framework-4-3-goes-rc1).

To make it work, we need to add in the `application.properties` file the following property (otherwise an `UnsatisfiedDependencyException` will be thrown):

```properties
spring.main.allow-circular-references=true
```

When we do a "self-injection" with the `@Autowired` annotation, what we actually get is our proxy! In this case, we
can try to call a `@Transactional` method though the proxy and in this way we'll get a transaction. For that we've added a new `public` method
annotated with the `@Transactional` annotation.

Let's check the logs to see if it actually works:

```
2022-05-16 07:31:05.159 DEBUG 1526158 --- [main] i.e.spring.tx.management.MovieService: entityManager: null
2022-05-16 07:31:05.183 DEBUG 1526158 --- [main] i.e.spring.tx.management.MovieService: entityManager: Shared EntityManager proxy for target factory [org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean@27c53c32]
2022-05-16 07:31:05.195 DEBUG 1526158 --- [main] o.s.orm.jpa.JpaTransactionManager    : Creating new transaction with name [inc.evil.spring.tx.management.MovieService.doInit]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2022-05-16 07:31:05.208 DEBUG 1526158 --- [main] o.s.orm.jpa.JpaTransactionManager    : Opened new EntityManager [SessionImpl(266906347<open>)] for JPA transaction
2022-05-16 07:31:05.210 DEBUG 1526158 --- [main] o.s.orm.jpa.JpaTransactionManager    : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@42107318]
2022-05-16 07:31:05.228 DEBUG 1526158 --- [main] o.s.orm.jpa.JpaTransactionManager    : Initiating transaction commit
2022-05-16 07:31:05.228 DEBUG 1526158 --- [main] o.s.orm.jpa.JpaTransactionManager    : Committing JPA transaction on EntityManager [SessionImpl(266906347<open>)]
Hibernate: 
    insert 
    into
        movies
        (name, id) 
    values
        (?, ?)
2022-05-16 07:31:05.240 DEBUG 1526158 --- [main] o.s.orm.jpa.JpaTransactionManager    : Closing JPA EntityManager [SessionImpl(266906347<open>)] after transaction
```

Indeed, our `JPA` entity was successfully inserted into the database.

{{< admonition type=note title="Note" open=false >}}

When doing the proxy self-injection, at the moment when the `@PostConstruct` method is invoked, it is obvious that the proxy for
our `MovieService` is ready (since `@PostConstruct` methods are called after all of the bean's dependencies we're set).

What if we try to rewrite our example like this?:

```java
@Service
@Slf4j
class MovieService {
    @Autowired
    private EntityManager entityManager;

    @Autowired
    private MovieService proxy;

    public MovieService() {
        log.debug("entityManager: {}", entityManager);
    }

    @PostConstruct
    @Transactional
    public void init() {
        log.debug("entityManager: {}", entityManager);
        Movie movie = Movie.builder()
                .name("Joker")
                .build();
        entityManager.persist(movie);
    }
}
```

Unfortunately it still won't work because of the way the `CommonAnnotationBeanPostProcessor` was implemented (well, to be more precise
its superclass - the `InitDestroyAnnotationBeanPostProcessor`), and it is the one
which is calling the `@PostConstruct` methods. This `BeanPostProcessor` use the original bean instance when invoking init-methods, not the proxy!

```java
package org.springframework.beans.factory.annotation;

public class InitDestroyAnnotationBeanPostProcessor
        implements DestructionAwareBeanPostProcessor, MergedBeanDefinitionPostProcessor, PriorityOrdered, Serializable {
    //...
    @Override //Here, the bean parameter is the original bean instance, not the proxy!
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
        try {
            metadata.invokeInitMethods(bean, beanName);
        }
        catch (InvocationTargetException ex) {
            throw new BeanCreationException(beanName, "Invocation of init method failed", ex.getTargetException());
        }
        catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Failed to invoke init method", ex);
        }
        return bean;
    }
    //...
```

{{< /admonition >}}

## Other pitfalls like this

When the `@PostConstruct` methods are called, any proxy-based mechanisms (like `@Async`, `@Secured`, `@Cacheable`) do not work, but
the fixes we've discussed in this blog post can be applicable.

For example, if we try to make the `@PostConstruct` method asynchronous, like this:

```java
@Slf4j
@SpringBootApplication
@EnableAsync
public class SpringDeclarativeTxManagementApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
    }
}

@Service
@Slf4j
class MovieService {
    @PostConstruct
    @Async
    public void init() {
        log.debug("Initializing MovieService");
    }
}
```

It won't work the way we expect it to. The `MovieService.init()` will be called from the `main` thread, not in a different one. See the logs below:

```
2022-05-16 08:36:36.779 DEBUG 1532607 --- [main] i.e.spring.tx.management.MovieService    : Initializing MovieService
```

But if we try to apply the trick [Listening to the ContextRefreshedEvent event](https://softice.dev/posts/spring_puzzler_transactional_poostconstruct_methods/#listening-to-the-contextrefreshedevent-event),
everything works as expected:

```java
@Slf4j
@SpringBootApplication
@EnableAsync
public class SpringDeclarativeTxManagementApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringDeclarativeTxManagementApplication.class, args);
    }
}

@Service
@Slf4j
class MovieService {

    @EventListener(ContextRefreshedEvent.class)
    @Async
    public void init() {
        log.debug("Initializing MovieService");
    }
}
```

By looking at the logs we can see that this time, the `MovieService.init()` method was called from the `task-1` thread.

```
2022-05-16 08:38:55.242 DEBUG 1532809 --- [task-1] i.e.spring.tx.management.MovieService    : Initializing MovieService
```

## Conclusion

In this blog post we've looked at one limitation of Spring's declarative transaction management - the fact that it can't be used
in `@PostConstruct` methods, since the proxy is not ready yet at that point in time.

We also looked at a couple of possible fixes to this problem, like using the programmatic approach, 
doing proxy self-injection or listening to the `ContextRefreshedEvent`.

Finally, we've discussed that this problem can be encountered when using other proxy-based mechanisms like `@Async`, `@Secured` or even `@Cacheable`.

There are also a couple of more puzzlers regarding the `@Transactional` annotation, we'll take a look at them in another blog post.

The code can be found on [GitHub](https://github.com/anrosca/spring_puzzler_transactional_PostConstruct_methods)

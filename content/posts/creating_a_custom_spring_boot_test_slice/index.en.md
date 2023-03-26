---
title: "Creating a custom Spring Boot test slice"
date: 2023-03-26T16:41:00+03:00
draft: false
author: "Andrei Rosca"
tags: ["Spring Boot", "Testing", "Localstack", "DynamoDB"]
categories: ["Spring Boot", "DynamoDB"]
---

## Introduction

In `Spring Boot`, when writing tests there's a way to slice up the test's application context, so that it contains only the beans which are appropriate
for the given test. Some examples are `@WebMvcTest`, `@DataJpaTest`, `@RestClientTest` and many others. For example, when testing
a jpa repository, we're not interested in the web-related components (like controllers), so using the `@DataJpaTest` will reduce the size
of the application context, so that it contains only the repositories and other infrastructure related to that (like `DataSource`s, `EntityManagerFactory` and others).

### When custom test-sliced are needed

Let's say we are working on an application which uses `DynamoDB`. It is a simple `REST API` for a simple anonymous forum, having categories, topics and comments.
Let's have a look more closely at the categories part of the application. The `Category` entity will look something like this:

```java
@Data
public class DynamoDbBase {
    protected String partitionKey;
    protected String sortKey;

    @DynamoDbPartitionKey
    @DynamoDbAttribute("PK")
    public String getPartitionKey() {
        return partitionKey;
    }

    @DynamoDbSortKey
    @DynamoDbAttribute("SK")
    public String getSortKey() {
        return sortKey;
    }
}
```

The `DynamoDbBase` will act as the base-class for all entities. It has a generic partition and sort key, because we plan to use a single `DynamoDB` table for
all entities (also known as `Single table design`).

Now the `Category` entity finally looks like the following:

```java
@DynamoDbBean
@EqualsAndHashCode(callSuper = true)
public class Category extends DynamoDbBase {
    public static final String CATEGORY_PK = "Category";
    public static final String CATEGORY_SK_PREFIX = "Category#";

    private String name;

    public Category() {
    }

    public Category(CategoryBuilder builder) {
        this.name = builder.name;
        this.partitionKey = builder.partitionKey;
        this.sortKey = builder.sortKey;
    }

    public void setId(String id) {
        setPartitionKey(CATEGORY_PK);
        setSortKey(CategoryKeyBuilder.makeSortKey(id));
    }

    public String getId() {
        return getSortKey().substring(CATEGORY_SK_PREFIX.length());
    }

    public String getName() {
        return this.name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public static CategoryBuilder builder() {
        return new CategoryBuilder();
    }

    public static class CategoryKeyBuilder {
        public static String makePartitionKey(String id) {
            return CATEGORY_PK;
        }

        public static String makeSortKey(String id) {
            return CATEGORY_SK_PREFIX + id;
        }
    }

    public static class CategoryBuilder {
        private String name;
        private String partitionKey;
        private String sortKey;

        public CategoryBuilder name(String name) {
            this.name = name;
            return this;
        }

        public CategoryBuilder partitionKey(String partitionKey) {
            this.partitionKey = partitionKey;
            return this;
        }

        public CategoryBuilder sortKey(String sortKey) {
            this.sortKey = sortKey;
            return this;
        }

        public Category build() {
            return new Category(this);
        }
    }
}
```

It is quite simple, it just has a `name` attribute, getters & setters and a builder.

Also we'll need to add a bit of configuration, by providing the `DynamoDbEnhancedClient` and `DynamoDbTable` beans, like shown below:

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(AwsProperties.class)
public class DynamoDbConfig {

    @Bean
    public DynamoDbClient dynamoDbClient(AwsProperties properties, AwsBasicCredentials awsCredentials) {
        var builder = DynamoDbClient.builder()
                                    .region(Region.of(properties.region()));
        if (properties.endpointOverride() != null) {
            builder.endpointOverride(URI.create(properties.endpointOverride()));
        }
        return builder.credentialsProvider(() -> awsCredentials).build();
    }

    @Bean
    public AwsBasicCredentials awsCredentials(AwsProperties properties) {
        return AwsBasicCredentials.create(properties.credentials().accessKey(), properties.credentials().secretKey());
    }

    @Bean
    public DynamoDbEnhancedClient dynamoDbEnhancedClient(DynamoDbClient dynamoDbClient) {
        return DynamoDbEnhancedClient.builder()
                                     .dynamoDbClient(dynamoDbClient)
                                     .build();
    }

    @Bean
    public DynamoDbTable<Category> categoryTable(DynamoDbEnhancedClient dynamoDbEnhancedClient, AwsProperties properties) {
        return dynamoDbEnhancedClient.table(properties.dynamoDbTableName(), TableSchema.fromBean(Category.class));
    }
}
```

Let's try to write a repository for the `Category` entity:

```java
public interface CategoryRepository {
    List<Category> findAll();

    Category create(Category categoryToCreate);

    Optional<Category> findById(String id);
}
```

The `CategoryRepository` interface specifies the repository contract, at the moment having the `findAll()`, `findById()` and `create()` operations. The implementation
can look something like this:

```java
@Repository
public class DynamoDBCategoryRepository implements CategoryRepository {

    private final DynamoDbTable<Category> categoryTable;

    public DynamoDBCategoryRepository(DynamoDbTable<Category> categoryTable) {
        this.categoryTable = categoryTable;
    }

    @Override
    public List<Category> findAll() {
        Key key = Key.builder()
                     .partitionValue(Category.CATEGORY_PK)
                     .sortValue(Category.CATEGORY_SK_PREFIX)
                     .build();
        return categoryTable.query(QueryConditional.sortBeginsWith(key))
                            .stream()
                            .flatMap(page -> page.items().stream())
                            .toList();
    }

    @Override
    public Category create(Category categoryToCreate) {
        categoryTable.putItem(categoryToCreate);
        return categoryToCreate;
    }

    @Override
    public Optional<Category> findById(String id) {
        Key key = Key.builder()
                     .partitionValue(Category.CategoryKeyBuilder.makePartitionKey(id))
                     .sortValue(Category.CategoryKeyBuilder.makeSortKey(id))
                     .build();
        return Optional.ofNullable(categoryTable.getItem(key));
    }
}
```

### Testing time

Now let's try to write a test for it. We'll end up with something like this:

```java
@SpringBootTest
public class DynamoDBCategoryRepositoryTest extends AbstractDatabaseTest {

    @Autowired
    private DynamoDbTable<Category> categoryTable;

    @Autowired
    private DynamoDBCategoryRepository categoryRepository;

    @BeforeEach
    public void setUp() {
        categoryTable.deleteTable();
        categoryTable.createTable();
        categoryTable.putItem(Category.builder()
                                      .name("Software development")
                                      .partitionKey("Category")
                                      .sortKey("Category#501735c3-5da7-4684-82d3-37af5d5dc44f")
                                      .build());
        categoryTable.putItem(Category.builder()
                                      .name("Anime")
                                      .partitionKey("Category")
                                      .sortKey("Category#601735c3-6da7-4684-62d3-47af5d5dc44e")
                                      .build());

    }

    @Test
    public void shouldBeAbleToFindAllCategories() {
        List<Category> expectedCategories = List.of(
            Category.builder()
                    .name("Software development")
                    .partitionKey("Category")
                    .sortKey("Category#501735c3-5da7-4684-82d3-37af5d5dc44f")
                    .build(),
            Category.builder()
                    .name("Anime")
                    .partitionKey("Category")
                    .sortKey("Category#601735c3-6da7-4684-62d3-47af5d5dc44e")
                    .build()
        );

        List<Category> actualCategories = categoryRepository.findAll();

        assertThat(actualCategories).isEqualTo(expectedCategories);
    }

    @Test
    public void shouldBeAbleToFindCategoriesById() {
        Category expectedCategory = Category.builder()
                                            .name("Software development")
                                            .partitionKey("Category")
                                            .sortKey("Category#501735c3-5da7-4684-82d3-37af5d5dc44f")
                                            .build();

        Optional<Category> actualCategory = categoryRepository.findById("501735c3-5da7-4684-82d3-37af5d5dc44f");

        assertThat(actualCategory).isEqualTo(Optional.of(expectedCategory));
    }
}
```

The `DynamoDBCategoryRepositoryTest` is quite simple. What's worth noting is that before each test we try to clean up the table, by deleting the table and 
re-creating it. We also insert some test data so that we have something to work with. Also it's worth pointing out that this test uses `Localstack` to
simulate the real `DynamoDB` service. We spin-up `Localstack` using `Testcontainers`, and the `AbstractDatabaseTest` contains all the guts related to that:

```java
public class AbstractDatabaseTest {

    private static final String TABLE_NAME = "forum";

    public static final LocalStackContainer localstack =
        new LocalStackContainer(DockerImageName.parse("localstack/localstack:1.4"))
            .withServices(LocalStackContainer.Service.DYNAMODB);

    @DynamicPropertySource
    public static void replaceProperties(DynamicPropertyRegistry registry) {
        registry.add("aws.endpoint-override", () -> localstack.getEndpointOverride(LocalStackContainer.Service.DYNAMODB));
        registry.add("aws.credentials.secret-key", localstack::getSecretKey);
        registry.add("aws.credentials.access-key", localstack::getAccessKey);
        registry.add("aws.region", localstack::getRegion);
    }

    static {
        localstack.start();
        createResources();
    }

    private static void createResources() {
        try {
            localstack.execInContainer("awslocal", "dynamodb", "create-table", "--table-name", TABLE_NAME,
                "--attribute-definitions", "AttributeName=PK,AttributeType=S", "AttributeName=SK,AttributeType=S",
                "--key-schema", "AttributeName=PK,KeyType=HASH", "AttributeName=SK,KeyType=RANGE",
                "--provisioned-throughput", "ReadCapacityUnits=5,WriteCapacityUnits=5");
        } catch (IOException | InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

A slight problem with this test is that it uses the `@SpringBootTest` which pretty much spins up the whole application context, containing not only
repository-specific classes, but also web related ones for example. This can increase the test's startup time and it's a shame, since we don't care about the
web layer for this particular test. Let's try to fix this, by writing a custom test slice.

## Creating a custom test-slice

It'll be nice to create a custom test-slice. We want something like `@DataJpaTest`, which allows us to either spin up only the repository layer. 
Let's take a look at the `@DataJpaTest` annotation, so that we find some inspiration:

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@BootstrapWith(DataJdbcTestContextBootstrapper.class)
@ExtendWith({SpringExtension.class})
@OverrideAutoConfiguration(
    enabled = false
)
@TypeExcludeFilters({DataJdbcTypeExcludeFilter.class})
@Transactional
@AutoConfigureCache
@AutoConfigureDataJdbc
@AutoConfigureTestDatabase
@ImportAutoConfiguration
public @interface DataJdbcTest {
    //...
}
```

The most interesting parts are the following:

- `@OverrideAutoConfiguration(enabled = false)`: disables auto-configuration
- `@BootstrapWith(DataJdbcTestContextBootstrapper.class)`: specifies how to bootstrap the application context. `DataJdbcTestContextBootstrapper` 
is a small specialization of `SpringBootTestContextBootstrapper`
- `@ImportAutoConfiguration`: allows to import some auto-configurations. It's useful since all auto-configurations were disabled by `@OverrideAutoConfiguration(enabled = false)`
- `@TypeExcludeFilters({DataJdbcTypeExcludeFilter.class})`: this is the most interesting one, this is the actual filter of the beans which we want to include/exclude
 from the application context

Everything else just configures the repository infrastructure, like enabling caching (via `@AutoConfigureCache`) and transactions (via `@Transactional`).

Now let's try to create a similar thing for `DynamoDB`. We'll create a custom annotation named `@DynamoDbTest`, which initially will look like the following:

```java
@ExtendWith(SpringExtension.class)
@OverrideAutoConfiguration(enabled = false)
@BootstrapWith(SpringBootTestContextBootstrapper.class)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface DynamoDbTest {
    Class<?>[] repositories() default {};
}
```

Since we didn't add yet the `@TypeExcludeFilters` annotation, with a custom implementation of the `TypeExcludeFilter`, we'll still end up with the whole application context.

But let's try to annotate our `DynamoDBCategoryRepositoryTest` with the `@DynamoDbTest` annotation, just to make sure everything still works:

```java
@DynamoDbTest
public class DynamoDBCategoryRepositoryTest extends AbstractDatabaseTest {

    @Autowired
    private DynamoDbTable<Category> categoryTable;

    @Autowired
    private DynamoDBCategoryRepository categoryRepository;

    @BeforeEach
    public void setUp() {
        categoryTable.deleteTable();
        categoryTable.createTable();
        categoryTable.putItem(Category.builder()
                                      .name("Software development")
                                      .partitionKey("Category")
                                      .sortKey("Category#501735c3-5da7-4684-82d3-37af5d5dc44f")
                                      .build());
        categoryTable.putItem(Category.builder()
                                      .name("Anime")
                                      .partitionKey("Category")
                                      .sortKey("Category#601735c3-6da7-4684-62d3-47af5d5dc44e")
                                      .build());

    }

    @Test
    public void shouldBeAbleToFindAllCategories() {
        List<Category> expectedCategories = List.of(
            Category.builder()
                    .name("Software development")
                    .partitionKey("Category")
                    .sortKey("Category#501735c3-5da7-4684-82d3-37af5d5dc44f")
                    .build(),
            Category.builder()
                    .name("Anime")
                    .partitionKey("Category")
                    .sortKey("Category#601735c3-6da7-4684-62d3-47af5d5dc44e")
                    .build()
        );

        List<Category> actualCategories = categoryRepository.findAll();

        assertThat(actualCategories).isEqualTo(expectedCategories);
    }
}
```

If we try to run the tests, they pass, which means that we didn't broke anything yet.

{{< figure src="tests-pass.png" alt="Tests are still passing" >}}

If we try to inject the `ApplicationContext` into our `DynamoDBCategoryRepositoryTest` and inspect it in the debugger, we'll see something like this:

{{< figure src="all-beans.png" alt="All beans" >}}

We can observe that our `ApplicationContext` contains all the beans, including the ones from the web-layer which we're not interested in for this particular test.

In order to fix that, we'll implement a custom `TypeExcludeFilter`. It will look something like this:

```java
public class DynamoDbTypeExcludeFilter extends AnnotationCustomizableTypeExcludeFilter {
    private static final Set<Class<?>> DEFAULT_INCLUDES = Set.of(Repository.class);

    private final DynamoDbTest annotation;

    public DynamoDbTypeExcludeFilter(Class<?> testClass) {
        this.annotation = AnnotatedElementUtils.getMergedAnnotation(testClass, DynamoDbTest.class);
    }

    @Override
    protected boolean hasAnnotation() {
        return this.annotation != null;
    }

    @Override
    protected boolean defaultInclude(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
        if (super.defaultInclude(metadataReader, metadataReaderFactory)) {
            return true;
        }
        for (Class<?> repository : this.annotation.repositories()) {
            if (isTypeOrAnnotated(metadataReader, metadataReaderFactory, repository)) {
                return true;
            }
        }
        return false;
    }

    @Override
    protected ComponentScan.Filter[] getFilters(FilterType type) {
        return new ComponentScan.Filter[0];
    }

    @Override
    protected boolean isUseDefaultFilters() {
        return true;
    }

    @Override
    protected Set<Class<?>> getDefaultIncludes() {
        return ObjectUtils.isEmpty(this.annotation.repositories()) ? DEFAULT_INCLUDES : Set.of();
    }

    @Override
    protected Set<Class<?>> getComponentIncludes() {
        return Set.of();
    }
}
```

This custom `TypeExcludeFilter` is tied to our `@DynamoDbTest` annotation.

The most important part is the following:

```java
private static final Set<Class<?>> DEFAULT_INCLUDES = Set.of(Repository.class);
```

Which means that by-default we'll include in the `ApplicationContext` only beans annotated with the `@Repository` annotation. If we take another look at our
`@DynamoDbTest` annotation:

```java
//...
public @interface DynamoDbTest {
    Class<?>[] repositories() default {};
}
```

we can observe that we have a `repositories()` attribute, which allows us to spin up for the test either all repositories (in case the `repositories()` is empty)
or a particular one, by specifying it like this: `@DynamoDbTest(repositories = DynamoDBCategoryRepository.class)`.

If we'll leave the `repositories()` attribute empty (like the following `@DynamoDbTest(repositories = {})`, then all `@Repository`-annotated classes will end up
in the application context. `DynamoDbTypeExcludeFilter.getDefaultIncludes())` method defines this behavior:

```java
public class DynamoDbTypeExcludeFilter extends AnnotationCustomizableTypeExcludeFilter {
 private static final Set<Class<?>> DEFAULT_INCLUDES = Set.of(Repository.class);
 //...
    @Override
    protected Set<Class<?>> getDefaultIncludes() {
        return ObjectUtils.isEmpty(this.annotation.repositories()) ? DEFAULT_INCLUDES : Set.of();
    }
 //...
}
```

In case we do specify the desired repository, like this: `@DynamoDbTest(repositories = DynamoDBCategoryRepository.class)`, then 
`DynamoDbTypeExcludeFilter.defaultInclude()` method specifies that only the `DynamoDBCategoryRepository` class should be included.

```java
public class DynamoDbTypeExcludeFilter extends AnnotationCustomizableTypeExcludeFilter {
    //...
    @Override
    protected boolean defaultInclude(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
        if (super.defaultInclude(metadataReader, metadataReaderFactory)) {
           return true;
        }
        for (Class<?> repository : this.annotation.repositories()) {
           if (isTypeOrAnnotated(metadataReader, metadataReaderFactory, repository)) {
            return true;
           }
        }
        return false;
    }
    //...
}
```

Now we'll need to register the new `TypeExcludeFilter` like this:

```java
@ExtendWith(SpringExtension.class)
@OverrideAutoConfiguration(enabled = false)
@BootstrapWith(SpringBootTestContextBootstrapper.class)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@TypeExcludeFilters(DynamoDbTypeExcludeFilter.class) //<--- this one was added
public @interface DynamoDbTest {
    Class<?>[] repositories() default {};
}
```

If we'll try to run our test now, it'll fail with the following error:

{{< figure src="error.png" alt="The test fails" >}}

The problem is that the `DynamoDbTypeExcludeFilter` included in the app context only the `@Repository`-annotated classes and filtered out everything else,
including our `DynamoDbConfig` java-config class, which declares our `DynamoDB` table.

In order to fix that, we'll need to register the `DynamoDbConfig` in the `@DynamoDbTest` annotation, like this:

```java
@ExtendWith(SpringExtension.class)
@OverrideAutoConfiguration(enabled = false)
@ImportAutoConfiguration(dev.softice.slice.config.DynamoDbConfig.class) //<--- this one was added
@BootstrapWith(SpringBootTestContextBootstrapper.class)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@TypeExcludeFilters(DynamoDbTypeExcludeFilter.class)
public @interface DynamoDbTest {
    Class<?>[] repositories() default {};
}

```

At this point, not only our test will pass, but it's `ApplicationContext` will contain only `DynamoDB`-specific beans, like shown below:

{{< figure src="finally-tests-pass.png" alt="The test passes" >}}

## Conclusion

In this blog post we saw how to create a custom `Spring Boot` test slice for `DynamoDB`. We saw that it's pretty easy, we just need a custom annotation annotated 
with:
- `@OverrideAutoConfiguration(enabled = false)`: disables auto-configuration
- `@ImportAutoConfiguration(dev.softice.slice.config.DynamoDbConfig.class)`: import a specific auto-configuration class
- `@BootstrapWith(SpringBootTestContextBootstrapper.class)`: specifies the application-context bootstrapper
- `@TypeExcludeFilters(DynamoDbTypeExcludeFilter.class)`: registers a `TypeExcludeFilter` which filters out beans we're not interested in

The example code we used in this article can be found on [GitHub](https://github.com/anrosca/spring-http-interfaces).

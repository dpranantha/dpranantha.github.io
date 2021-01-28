---
title: "Kotlin coroutines and Java: How do they interoperate? - Part 1"
permalink: /kotlin-coroutines-java-interoperate-part-1
date: 2021-01-28 22:45:00+01:00
categories:
- Coding
tags:
- Kotlin
- Coroutine
- Spring
---

If we code with Kotlin, then Kotlin coroutine is a semi-obvious choice for asynchronous and concurrent programming. There are
other alternatives, such as:
- threading
- callbacks
- future
- reactive extensions, e.g., [rxJava](https://github.com/ReactiveX/RxJava) and [Spring Reactor](https://projectreactor.io/)

Each has its pros and cons, and [here](https://kotlinlang.org/docs/tutorials/coroutines/async-programming.html) describes it better.

In this writing, the question is not about using coroutines, threads, callbacks, etc. We are more interested in 
how Kotlin coroutines can interoperate with Java codes and [Spring Webclient](https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/html/boot-features-webclient.html) if you have such a case. 
It means the whole flow should be asynchronous and concurrent.

I assume you have some knowledge on threads vs. coroutines (so-called lightweight or green threads), asynchronous programming,
event-loop vs. thread-per-request, and concurrent vs. parallel processing.

First, let's describe a use-case. Suppose we have a legacy back-end project which calls several external APIs.
It constructs a response that combines the responses from each external API. An example of such a use-case is an aggregation service 
that fetches data and transforms them into a single API response. The codebase was written in Java,
and you want to migrate it into Kotlin. However, you don't want to migrate it all at once due to complexity. Moreover, the Java codes
are all synchronous `thread-per-request` service. Thus, you want to scale better by making them asynchronous. How do you approach it?

Breaking down the problem set, we need to
1. Migrate the codebase from Java to Kotlin one at a time
2. Make the codes behave asynchronously
3. Move away from the `thread-per-request` style of execution

We can set the objectives during migration
1. Existing unit tests should remain valid with no/little adjustment.
2. Additional unit tests are created to check concurrency.
3. Integration tests should remain valid. 

The figure below shows the architecture of our aggregation service example. The example simulates calls to other services using fixed delays with static data pre-populated in a cache.

![An aggregator service](/assets/images/kotlin-coroutine/cache-simulated-aggregator.jpg "An example of aggregator service for retrieving product summary")

Let's take a look at [the existing Java codebase](https://github.com/dpranantha/kotlin-coroutine-interops). Mind you that this is an oversimplified example for the sake of brevity. 
The example uses [Springboot](https://spring.io/projects/spring-boot) setup with [Apache Tomcat](http://tomcat.apache.org/) as the container.

The `AggregationService` class collects and aggregates data from its five data provider services sequentially.
1. Call `ProductCatalogueService` to retrieve product catalog information. If the response fails, then throw an exception.
2. Call `ProductDescriptionService` to retrieve **optional** product description information.
3. Call `ProductOfferService` to retrieve **optional** product offers information.
4. If the response from step 3 is not empty, call `SellerService` for each offer to retrieve seller information. Otherwise, go to 5.
5. Call `ProductReviewService` to retrieve **optional** product review information.
6. Aggregate retrieved information into a single response, called `ProductSummary`.

```java
@Service
public class AggregatorService {
    private final ProductCatalogueService productCatalogueService;
    private final ProductDescriptionService productDescriptionService;
    private final ProductOfferService offerService;
    private final SellerService sellerService;
    private final ProductReviewService reviewService;

    @Autowired
    public AggregatorService(ProductCatalogueService productCatalogueService,
                             ProductDescriptionService productDescriptionService,
                             ProductOfferService offerService,
                             SellerService sellerService,
                             ProductReviewService reviewService) {
         //truncated for brevity
    }

    public ProductSummary getProductSummary(String productId) throws ProductNotFoundException {
        // 1. get product
        final ProductCatalogue productCatalogue = productCatalogueService.getProductInfo(productId)
                .orElseThrow(() -> new ProductNotFoundException("Product can't be found!"));
        final Optional<ProductDescription> productDescription = productDescriptionService.getProductDescription(productId);
        final List<ProductOfferAndSeller> productOfferAndSellers = getProductOfferAndSellers(productId);
        final Pair<List<String>, Double> productReviews = getProductReviews(productId);
        return new ProductSummary(
           //truncated for brevity
        );
    }

    private List<ProductOfferAndSeller> getProductOfferAndSellers(String productId) {
        return offerService.getProductOffers(productId).stream()
                //truncated for brevity
                .collect(Collectors.toList());
    }

    private Pair<List<String>, Double> getProductReviews(String productId) {
        final List<ProductReview> reviews = reviewService.getReviews(productId);
        //truncated for brevity
        return Pair.of(allReviews, rating);
    }
}
```

The project has some unit tests and integration tests (refer to [README](https://github.com/dpranantha/kotlin-coroutine-interops/blob/main/README.md) 
for `how to run` instructions). The integration tests check both the content of the response, and the expected execution time.
In this case, the expected execution (processing) time is above 1200ms for a valid response. 

![Expected processing time](/assets/images/kotlin-coroutine/execution-time.png "Expected processing time")

However, the overall latency is larger. That also applies to the integration tests below. The calculation is the following:

1. Calling four data provider services: `ProductCatalogueService`, `ProductDescriptionService`, `ProductOfferService`, `ProductReviewService`,
   with each having a 200 ms delay, gives a total >= 800ms execution time.
2. Calling `SellerService` twice with a 200 ms delay (each product has two static offers) gives a total >= 400ms execution time.
3. Latency overhead is above 400ms. This in total gives around 1600 - 1800 ms.

```java
@SpringBootTest(classes = {CoroutineInteropsApplication.class},
        webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
class CoroutineInteropsIT {

   @Test
   void testExistsProduct_thenReceived_200_Response() {
      final ProductSummary productSummary = when().request("GET", "/v1/products/1")
              .then()
              .time(greaterThan(1600L), TimeUnit.MILLISECONDS)  //6 call x 200ms + with overhead >= 300ms
              .time(lessThan(1800L), TimeUnit.MILLISECONDS)
              .assertThat()
              .statusCode(200)
              .extract()
              .body()
              .as(ProductSummary.class);

      assertEquals("1", productSummary.getProductId());
      assertEquals("Product 1", productSummary.getProductName());
      assertEquals("This is product 1", productSummary.getProductDescription().get());
      assertEquals(1.0, productSummary.getProductWeightInKg().get());
      assertEquals("red", productSummary.getProductColor().get());
      assertEquals(Arrays.asList(22.99, 12.99), productSummary.getProductOfferAndSellers().stream()
              .map(ProductOfferAndSeller::getProductPrice)
              .collect(Collectors.toList()));
      assertEquals(Arrays.asList("Seller 1", "Seller 2"), productSummary.getProductOfferAndSellers().stream()
              .map(ProductOfferAndSeller::getSellerName)
              .filter(Optional::isPresent)
              .map(Optional::get)
              .collect(Collectors.toList()));
      assertEquals(4.0, productSummary.getRating());
   }

   @Test
   void testNotExistsProduct_thenReceived_404_Response() {
      when().request("GET", "/v1/products/1100")
              .then()
              .time(greaterThan(200L), TimeUnit.MILLISECONDS)
              .time(lessThan(300L), TimeUnit.MILLISECONDS)
              .assertThat()
              .statusCode(404)
              .body("message", equalTo("Product can't be found!"))
              .body("errorCode", equalTo(404));
   }
}
```

We plan to migrate the codebase to Kotlin one step at a time while adopting the Kotlin coroutine. Let's migrate
`ProductDescriptionService` first as an example. In this case, we expect to decrease the execution time by 200 ms.
First, we need to include Kotlin in our Maven dependencies, mainly `kotlin-stdlib`, `kotlinx-coroutines`, and `kotlin-maven-plugins`.
We can add `io.mockk`, a useful mocking library in Kotlin. [Github feature branch](https://github.com/dpranantha/kotlin-coroutine-interops/tree/feature/migrate-productdescription) 
shows the overall dependency setup, and the migrated codes.

Next step is to create kotlin source and test packages and port unit test for `AggregationService` (i.e., `AggregationServivceTests`) into Kotlin (as-is).
Thus, we are ready to introduce Kotlin codebase for `ProductDescriptionService`. Below shows both Java and Kotlin codebase for `ProductDescriptionService`.

```java
@Service
class ProductDescriptionService {
    private final DataProvider dataProvider;

    @Autowired
    ProductDescriptionService(DataProvider dataProvider) {
        this.dataProvider = dataProvider;
    }

    public Optional<ProductDescription> getProductDescription(String productId) {
        return dataProvider.getProductDescription(productId);
    }
}
```

```kotlin
@Service
open class ProductDescriptionServiceKt(private val dataProvider: DataProvider) {
    suspend fun getProductDescription(productId: String): ProductDescription? = coroutineScope {
        dataProvider.getProductDescription(productId).unwrap()
    }

    fun getProductDescriptionJavaCall(productId: String): CompletableFuture<Optional<ProductDescription>> = GlobalScope.future {
        getProductDescription(productId).wrap()
    }
}

fun <T> Optional<T>.unwrap(): T? = orElse(null)
fun <T> T?.wrap() = Optional.ofNullable(this)
```

The Kotlin codebase introduces coroutines with a `suspend` modifier and `coroutineScope.` It also introduces a method callable
from Java temporarily before we migrate the whole codebase to Kotlin. This uses `GlobalScope.future` construct which translates to `CompletableFuture`
in Java. We can opt to implement `dataProvider.getProductDescription(productId)` inside `getProductDescriptionJavaCall` method to avoid box/unboxing.
However, I want the real pipeline to execute the suspendable method.

The next step is to change `AggregatorService` to call `ProductDescriptionService` in both Java codebase and Kotlin codebase.
It introduces a flag to enable switching between them. Moreover, this allows a canary release setup for people with a slightly conservative approach.
The code below shows the changes compared to the original `AggregatorService` codebase above.

```java
@Service
public class AggregatorService {
    private static Logger logger = LoggerFactory.getLogger(AggregatorService.class);

    private final ProductCatalogService productCatalogService;
    private final ProductDescriptionService productDescriptionService;
    private final ProductOfferService offerService;
    private final SellerService sellerService;
    private final ProductReviewService reviewService;
    private final ProductDescriptionServiceKt productDescriptionServiceKt;  //additional injected Kotlin class
    private final boolean useKotlin; //switching flag

    @Autowired
    public AggregatorService(ProductCatalogService productCatalogService,
                             ProductDescriptionService productDescriptionService,
                             ProductOfferService offerService,
                             SellerService sellerService,
                             ProductReviewService reviewService,
                             ProductDescriptionServiceKt productDescriptionServiceKt,
                             @Value("${use.kotlin:false}") boolean useKotlin) {
        this.productCatalogService = productCatalogService;
        this.productDescriptionService = productDescriptionService;
        this.offerService = offerService;
        this.sellerService = sellerService;
        this.reviewService = reviewService;
        this.productDescriptionServiceKt = productDescriptionServiceKt;
        this.useKotlin = useKotlin;
    }

    public ProductSummary getProductSummary(String productId) throws ProductNotFoundException {
        final ProductCatalog productCatalog = productCatalogService.getProductCatalog(productId)
                .orElseThrow(() -> new ProductNotFoundException("Product can't be found!"));
        final CompletableFuture<Optional<ProductDescription>> productDescriptionAsync = getProductDescriptionReroute(productId)
                .exceptionally(t -> {
                    logger.error("Error retrieving data for product description {}", t.getCause().getLocalizedMessage());
                    return Optional.empty();
                });  //the call change
        final List<ProductOfferAndSeller> productOfferAndSellers = getProductOfferAndSellers(productId);
        final Pair<List<String>, Double> productReviews = getProductReviews(productId);
        final Optional<ProductDescription> productDescription = productDescriptionAsync.join(); //blocking future
        return new ProductSummary(
                //truncated for brevity
                );
    }

    //additional method to reroute getting product description based on the injected flag
    private CompletableFuture<Optional<ProductDescription>> getProductDescriptionReroute(String productId) {
        if (useKotlin) {
            return productDescriptionServiceKt.getProductDescriptionJavaCall(productId);
        } else  {
            return CompletableFuture.supplyAsync(() -> productDescriptionService.getProductDescription(productId));
        }
    }

    private List<ProductOfferAndSeller> getProductOfferAndSellers(String productId) {
        return offerService.getProductOffers(productId).stream()
                //truncated for brevity
                .collect(Collectors.toList());
    }

    private Pair<List<String>, Double> getProductReviews(String productId) {
        final List<ProductReview> reviews = reviewService.getReviews(productId);
       //truncated for brevity
        return Pair.of(allReviews, rating);
    }
}
```

There are only a few changes. The rerouting function reroutes the call to either Kotlin or Java codebase.
For simplicity, the code performs **blocking** on `CompletableFuture` by invoking `join` method. A fair
question would be: 

* I think Java codebase with `CompletableFuture.supplyAsync` wrapper will perform better 
since it has less overhead of starting coroutines. So why bother?

Our goal is to migrate to codebase from Java to Kotlin while improving performance using Kotlin 
concurrency toolset (read: coroutines). You can also port the Java codebase to Kotlin and improve it later. 
However, this blog also intends to give you an overview of coroutine interoperability with Java.

Next, we need to modify the unit tests for `AggregatorService` in both Java and Kotlin ported one shown below.
In case you have a question over the `open` class signature in `ProductDescriptionServiceKt,` the answer is that
we need to mock it in Java using `Mockito.` There are other ways to mock a `final` Kotlin class in Java. 
I leave it to readers to figure it out themselves.

```java
@RunWith(MockitoJUnitRunner.class)
public class AggregatorServiceTests {
    @Mock
    private ProductCatalogService mockProductCatalogService;
    @Mock
    private ProductDescriptionService mockProductDescriptionService;
    @Mock
    private ProductOfferService mockOfferService;
    @Mock
    private SellerService mockSellerService;
    @Mock
    private ProductReviewService mockReviewService;
    @Mock
    private ProductDescriptionServiceKt mockProductDescriptionServiceKt; //additional field for Kotlin codebase ProductDescriptionService

    private AggregatorService aggregatorService;

    @Before
    public void setup(){
        aggregatorService = new AggregatorService(
                mockProductCatalogService,
                mockProductDescriptionService,
                mockOfferService,
                mockSellerService,
                mockReviewService,
                mockProductDescriptionServiceKt,
                false);  //flag = false to call Java codebase. Thus, the logic would have been the same
    }

    @Test
    public void givenAllValidData_ThenReturnsProductSummary() throws ProductNotFoundException {
        Mockito.when(mockProductCatalogService.getProductCatalog(anyString()))
                .thenReturn(Optional.of(new ProductCatalog("1", "razor x1")));
        Mockito.when(mockProductDescriptionService.getProductDescription(anyString()))
                .thenReturn(Optional.of(new ProductDescription("1", "this is a razor x1", 1.5, "silver")));
        Mockito.when(mockOfferService.getProductOffers(anyString()))
                .thenReturn(Arrays.asList(new ProductOffer("1", 20.0, "s-1"),
                        new ProductOffer("2", 19.9, "s-2")));
        Mockito.when(mockSellerService.getSeller("s-1"))
                .thenReturn(Optional.of(new Seller("s-1", "expensive seller")));
        Mockito.when(mockSellerService.getSeller("s-2"))
                .thenReturn(Optional.of(new Seller("s-2", "just a seller")));
        Mockito.when(mockReviewService.getReviews(anyString()))
                .thenReturn(Arrays.asList(new ProductReview("1", "anonymous", "that is awesome", 5),
                        new ProductReview("2", "mr. A", "that is ok", 3)));

        final ProductSummary productSummary = aggregatorService.getProductSummary("1");
        Assertions.assertEquals("razor x1", productSummary.getProductName());
        Assertions.assertEquals("silver", productSummary.getProductColor().get());
        Assertions.assertEquals("this is a razor x1", productSummary.getProductDescription().get());
        Assertions.assertEquals(1.5, productSummary.getProductWeightInKg().get());
        Assertions.assertEquals(2, productSummary.getProductOfferAndSellers().size());
        Assertions.assertEquals("expensive seller", productSummary.getProductOfferAndSellers().get(0).getSellerName().get());
        Assertions.assertEquals(20.0, productSummary.getProductOfferAndSellers().get(0).getProductPrice());
        Assertions.assertEquals("just a seller", productSummary.getProductOfferAndSellers().get(1).getSellerName().get());
        Assertions.assertEquals(19.9, productSummary.getProductOfferAndSellers().get(1).getProductPrice());
        Assertions.assertEquals(2, productSummary.getProductReviews().size());
        Assertions.assertEquals(Arrays.asList("that is awesome", "that is ok"), productSummary.getProductReviews());
        Assertions.assertEquals(4.0, productSummary.getRating());
    }

    @Test(expected = ProductNotFoundException.class)
    public void givenNotFoundProduct_ThrowsException() throws ProductNotFoundException {
        Mockito.when(mockProductCatalogService.getProductCatalog(anyString()))
                .thenReturn(Optional.empty());

        aggregatorService.getProductSummary("1");
    }

    @Test
    public void givenOnlyValidProductCatalogAndOfferPriceData_ThenReturnsProductSummary() throws ProductNotFoundException {
        Mockito.when(mockProductCatalogService.getProductCatalog(anyString()))
                .thenReturn(Optional.of(new ProductCatalog("1", "razor x1")));
        Mockito.when(mockProductDescriptionService.getProductDescription(anyString()))
                .thenReturn(Optional.empty());
        Mockito.when(mockOfferService.getProductOffers(anyString()))
                .thenReturn(Collections.singletonList(new ProductOffer("1", 20.0, "s-1")));
        Mockito.when(mockSellerService.getSeller("s-1"))
                .thenReturn(Optional.empty());
        Mockito.when(mockReviewService.getReviews(anyString()))
                .thenReturn(Collections.emptyList());

        final ProductSummary productSummary = aggregatorService.getProductSummary("1");
        Assertions.assertEquals("razor x1", productSummary.getProductName());
        Assertions.assertFalse(productSummary.getProductColor().isPresent());
        Assertions.assertFalse(productSummary.getProductDescription().isPresent());
        Assertions.assertFalse(productSummary.getProductWeightInKg().isPresent());
        Assertions.assertEquals(1, productSummary.getProductOfferAndSellers().size());
        Assertions.assertFalse(productSummary.getProductOfferAndSellers().get(0).getSellerName().isPresent());
        Assertions.assertEquals(20.0, productSummary.getProductOfferAndSellers().get(0).getProductPrice());
        Assertions.assertTrue(productSummary.getProductReviews().isEmpty());
        Assertions.assertEquals(0.0, productSummary.getRating());
    }
}
```


```kotlin
class AggregatorServiceKtTests {
    private val mockProductCatalogService: ProductCatalogService = mockk()
    private val mockProductDescriptionService: ProductDescriptionService = mockk()
    private val mockOfferService: ProductOfferService = mockk()
    private val mockSellerService: SellerService = mockk()
    private val mockReviewService: ProductReviewService = mockk()
    private val mockProductDescriptionServiceKt: ProductDescriptionServiceKt = mockk()

    private lateinit var aggregatorService: AggregatorService

    @BeforeEach
    fun setup() {
        aggregatorService = AggregatorService(
            mockProductCatalogService,
            mockProductDescriptionService,
            mockOfferService,
            mockSellerService,
            mockReviewService,
            mockProductDescriptionServiceKt,
            true  //flag = true to call Kotlin codebase `ProductDescriptionServiceKt`
        )
    }

    @Test
    @Throws(ProductNotFoundException::class)
    fun givenAllValidData_ThenReturnsProductSummary() {
        every { mockProductCatalogService.getProductCatalog(any()) } returns Optional.of(ProductCatalog("1", "razor x1"))
        every { mockProductDescriptionServiceKt.getProductDescriptionJavaCall(any()) } returns CompletableFuture.supplyAsync { Optional.of(ProductDescription("1", "this is a razor x1", 1.5, "silver")) }
        every { mockOfferService.getProductOffers(any()) } returns listOf(
                    ProductOffer("1", 20.0, "s-1"),
                    ProductOffer("2", 19.9, "s-2")
                )
        every { mockSellerService.getSeller("s-1") } returns Optional.of(Seller("s-1", "expensive seller"))
        every { mockSellerService.getSeller("s-2") } returns Optional.of(Seller("s-2", "just a seller"))
        every { mockReviewService.getReviews(any()) }  returns listOf(
                    ProductReview("1", "anonymous", "that is awesome", 5),
                    ProductReview("2", "mr. A", "that is ok", 3)
                )

        val productSummary = aggregatorService.getProductSummary("1")

        Assertions.assertEquals("razor x1", productSummary.productName)
        Assertions.assertEquals("silver", productSummary.productColor.get())
        Assertions.assertEquals("this is a razor x1", productSummary.productDescription.get())
        Assertions.assertEquals(1.5, productSummary.productWeightInKg.get())
        Assertions.assertEquals(2, productSummary.productOfferAndSellers.size)
        Assertions.assertEquals("expensive seller", productSummary.productOfferAndSellers[0].sellerName.get())
        Assertions.assertEquals(20.0, productSummary.productOfferAndSellers[0].productPrice)
        Assertions.assertEquals("just a seller", productSummary.productOfferAndSellers[1].sellerName.get())
        Assertions.assertEquals(19.9, productSummary.productOfferAndSellers[1].productPrice)
        Assertions.assertEquals(2, productSummary.productReviews.size)
        Assertions.assertEquals(Arrays.asList("that is awesome", "that is ok"), productSummary.productReviews)
        Assertions.assertEquals(4.0, productSummary.rating)
    }

    @Test
    @Throws(ProductNotFoundException::class)
    fun givenNotFoundProduct_ThrowsException() {
        every { mockProductCatalogService.getProductCatalog(any()) } returns Optional.empty()

        Assertions.assertThrows(ProductNotFoundException::class.java) { aggregatorService.getProductSummary("1") }
    }

    @Test
    @Throws(ProductNotFoundException::class)
    fun givenOnlyValidProductCatalogAndOfferPriceData_ThenReturnsProductSummary() {
        every { mockProductCatalogService.getProductCatalog(any()) } returns Optional.of(ProductCatalog("1", "razor x1"))
        every { mockProductDescriptionServiceKt.getProductDescriptionJavaCall(any()) } returns CompletableFuture.supplyAsync { Optional.empty() }
        every { mockOfferService.getProductOffers(any()) } returns listOf(ProductOffer("1", 20.0, "s-1"))
        every { mockSellerService.getSeller("s-1") } returns Optional.empty()
        every { mockReviewService.getReviews(any()) } returns emptyList()

        val productSummary = aggregatorService.getProductSummary("1")

        Assertions.assertEquals("razor x1", productSummary.productName)
        Assertions.assertFalse(productSummary.productColor.isPresent)
        Assertions.assertFalse(productSummary.productDescription.isPresent)
        Assertions.assertFalse(productSummary.productWeightInKg.isPresent)
        Assertions.assertEquals(1, productSummary.productOfferAndSellers.size)
        Assertions.assertFalse(productSummary.productOfferAndSellers[0].sellerName.isPresent)
        Assertions.assertEquals(20.0, productSummary.productOfferAndSellers[0].productPrice)
        Assertions.assertTrue(productSummary.productReviews.isEmpty())
        Assertions.assertEquals(0.0, productSummary.rating)
    }
}
```

The unit-test changes are minimal in the Java codebase (a switch flag `false` and mocked `mockProductDescriptionServiceKt`). The
tests should just run as-is. The Kotlin ported unit tests are very much alike. The only difference is that
the switch flag is `true.` If we run both versions, they will pass.

We also need to adjust the integration tests. Guess what? the only adjustments are the flag and reducing the latency value since we
perform `getProductDescription` concurrently.

```java
@SpringBootTest(classes = {CoroutineInteropsApplication.class},
        webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT,
        properties = { "use.kotlin=false" } ) //override application.properties to still use Java codebase
class CoroutineInteropsIT {

    @Test
    void testExistsProduct_thenReceived_200_Response() {
        final ProductSummary productSummary = when().request("GET", "/v1/products/1")
                .then()
                .time(greaterThan(1400L), TimeUnit.MILLISECONDS)  //change: 6 call x 200ms (1 concurrent) + with overhead >= 300ms
                .time(lessThan(1600L), TimeUnit.MILLISECONDS)
                .assertThat()
                .statusCode(200)
                .extract()
                .body()
                .as(ProductSummary.class);

        assertEquals("1", productSummary.getProductId());
        assertEquals("Product 1", productSummary.getProductName());
        assertEquals("This is product 1", productSummary.getProductDescription().get());
        assertEquals(1.0, productSummary.getProductWeightInKg().get());
        assertEquals("red", productSummary.getProductColor().get());
        assertEquals(Arrays.asList(22.99, 12.99), productSummary.getProductOfferAndSellers().stream()
                .map(ProductOfferAndSeller::getProductPrice)
                .collect(Collectors.toList()));
        assertEquals(Arrays.asList("Seller 1", "Seller 2"), productSummary.getProductOfferAndSellers().stream()
                .map(ProductOfferAndSeller::getSellerName)
                .filter(Optional::isPresent)
                .map(Optional::get)
                .collect(Collectors.toList()));
        assertEquals(4.0, productSummary.getRating());
    }

    @Test
    void testNotExistsProduct_thenReceived_404_Response() {
        when().request("GET", "/v1/products/1100")
                .then()
                .time(greaterThan(200L), TimeUnit.MILLISECONDS)
                .time(lessThan(300L), TimeUnit.MILLISECONDS)
                .assertThat()
                .statusCode(404)
                .body("message", equalTo("Product can't be found!"))
                .body("errorCode", equalTo(404));
    }
}
```

Another good question: what does happen if we set the `use.kotlin` flag to `true`?

It will have an overhead of the same amount of latency as the sequential call, **at least for integration tests**.
However, if we run the application, it will give you the same processing time (and thus latency), around 1000 ms.

`use.kotlin` = `false`

![Kotlin flag false](/assets/images/kotlin-coroutine/kotlin-flag-false.png "processing time for Kotlin flag `false`")

`use.kotlin` = `true`

![Kotlin flag true](/assets/images/kotlin-coroutine/kotlin-flag-true.png "processing time for Kotlin flag `true`")

In the next part, we will discuss refactoring `AggregateService` into Kotlin codebase and make the whole flow 
becoming asynchronous non-blocking Kotlin codebase.

Have fun coding!

---
title: "Kotlin coroutines and Java: How do they interoperate? - Part 2"
permalink: /kotlin-coroutines-java-interoperate-part-2
date: 2021-03-01 16:55:00+01:00
categories:
- Coding
tags:
- Kotlin
- Coroutine
- Spring
---

This post continues what we have left off last time when migrating [Java blocking codebase into Kotlin coroutine and how they interoperate
with each other]({% post_url 2021/2021-01-18-kotlin-coroutines %}). To summarize, we have migrated `ProductDescriptionService` from Java to Kotlin
and enabled it in `AggregateService` via a `useKotlin` switch. Moreover, we have created unit tests in Kotlin version, whereas the existing Java version
has some minor adjustments. The timing in integration tests is adjusted as we're moving towards asynchronous (and concurrent) programming.

The figure below serves as reminder of our aggregation service architecture.

![An aggregator service](/assets/images/kotlin-coroutine/cache-simulated-aggregator.jpg "An example of aggregator service for retrieving product summary")

Once we understand the pattern for migration, we can continue migrating all service calls from Java to Kotlin and enabled it in `AggregateService` via
the switch. The `AggregationService` code below shows one possible state when we have successfully migrated them. The `AggregatorService` code is available at 
the [Github feature branch](https://github.com/dpranantha/kotlin-coroutine-interops/blob/feature/migrate-all-calls-except-aggregator/src/main/java/com/dpranantha/coroutineinterops/service/AggregatorService.java).

```java
@Service
public class AggregatorService {
   private static final Logger logger = LoggerFactory.getLogger(AggregatorService.class);

   //Java
   private final ProductCatalogService productCatalogService;
   private final ProductDescriptionService productDescriptionService;
   private final ProductOfferService offerService;
   private final SellerService sellerService;
   private final ProductReviewService reviewService;
   
   //Kotlin
   private final ProductCatalogServiceKt productCatalogServiceKt;
   private final ProductDescriptionServiceKt productDescriptionServiceKt;
   private final ProductOfferServiceKt offerServiceKt;
   private final SellerServiceKt sellerServiceKt;
   private final ProductReviewServiceKt reviewServiceKt;
   
   //Flag
   private final boolean useKotlin;

   @Autowired
   public AggregatorService(ProductCatalogService productCatalogService,
                            ProductDescriptionService productDescriptionService,
                            ProductOfferService offerService,
                            SellerService sellerService,
                            ProductReviewService reviewService,
                            ProductCatalogServiceKt productCatalogServiceKt,
                            ProductDescriptionServiceKt productDescriptionServiceKt,
                            ProductOfferServiceKt offerServiceKt,
                            SellerServiceKt sellerServiceKt,
                            ProductReviewServiceKt reviewServiceKt,
                            @Value("${use.kotlin:false}") boolean useKotlin) {
      this.productCatalogService = productCatalogService;
      this.productDescriptionService = productDescriptionService;
      this.offerService = offerService;
      this.sellerService = sellerService;
      this.reviewService = reviewService;
      this.productCatalogServiceKt = productCatalogServiceKt;
      this.productDescriptionServiceKt = productDescriptionServiceKt;
      this.offerServiceKt = offerServiceKt;
      this.sellerServiceKt = sellerServiceKt;
      this.reviewServiceKt = reviewServiceKt;
      this.useKotlin = useKotlin;
   }

   public ProductSummary getProductSummary(String productId) throws ProductNotFoundException {
      return getProductCatalogReroute(productId)
              .exceptionally(t -> {
                 logger.error("Error retrieving data for product catalog with productId {}: {}", productId, t.getCause().getLocalizedMessage());
                 return Optional.empty();
              })
              .thenApply(productCatalogOpt -> productCatalogOpt.map(
                      productCatalog -> {
                         final CompletableFuture<Optional<ProductDescription>> productDescriptionAsync = getProductDescriptionReroute(productId)
                                 .exceptionally(t -> {
                                    logger.error("Error retrieving data for product description with productId {}: {}", productId, t.getCause().getLocalizedMessage());
                                    return Optional.empty();
                                 });
                         final CompletableFuture<List<ProductOfferAndSeller>> productOfferAndSellersAsync = getProductOfferAndSellersReroute(productId)
                                 .exceptionally(t -> {
                                    logger.error("Error retrieving data for product offer with productId {}: {}", productId, t.getCause().getLocalizedMessage());
                                    return Collections.emptyList();
                                 });
                         final CompletableFuture<Pair<List<String>, Double>> productReviewsAsync = getProductReviewsReroute(productId)
                                 .exceptionally(t -> {
                                    logger.error("Error retrieving data for product review with productId {}: {}", productId, t.getCause().getLocalizedMessage());
                                    return Pair.of(Collections.emptyList(), 0d);
                                 });
                         final Optional<ProductDescription> productDescription = productDescriptionAsync.join();
                         final List<ProductOfferAndSeller> productOfferAndSellers = productOfferAndSellersAsync.join();
                         final Pair<List<String>, Double> productReviews = productReviewsAsync.join();
                         return new ProductSummary(productId,
                                 productCatalog.getProductName(),
                                 productDescription.map(ProductDescription::getShortDescription).orElse(null),
                                 productDescription.map(ProductDescription::getWeightInKg).orElse(null),
                                 productDescription.map(ProductDescription::getColor).orElse(null),
                                 productOfferAndSellers,
                                 productReviews.getLeft(),
                                 productReviews.getRight());
                      })
              )
              .join() //immediately blocking in this case
              .orElseThrow(() -> new ProductNotFoundException("Product can't be found!"));
   }

   private CompletableFuture<Optional<ProductCatalog>> getProductCatalogReroute(String productId) {
      if (useKotlin) {
         return productCatalogServiceKt.getProductCatalogJavaCall(productId);
      } else {
         return CompletableFuture.supplyAsync(() -> productCatalogService.getProductCatalog(productId));
      }
   }

   private CompletableFuture<Optional<ProductDescription>> getProductDescriptionReroute(String productId) {
      if (useKotlin) {
         return productDescriptionServiceKt.getProductDescriptionJavaCall(productId);
      } else  {
         return CompletableFuture.supplyAsync(() -> productDescriptionService.getProductDescription(productId));
      }
   }

   private CompletableFuture<List<ProductOfferAndSeller>> getProductOfferAndSellersReroute(String productId) {
      if (useKotlin) {
          //Note: This shows the challenges to construct CompletableFuture pipeline, Spring Reactor has slightly better operator naming for pipelining
         return offerServiceKt.getProductOffersForJavaCall(productId)
                 .thenCompose(productOffers -> {
                    final List<CompletableFuture<ProductOfferAndSeller>> productOfferAndSellerFutures = productOffers.stream()
                            .map(productOffer -> sellerServiceKt.getSellerForJavaCall(productOffer.getSellerId())
                                    .exceptionally(t -> {
                                       logger.error("Error retrieving data for seller with sellerId {}: {}", productOffer.getSellerId(), t.getCause().getLocalizedMessage());
                                       return Optional.empty();
                                    })
                                    .thenApply(seller -> new ProductOfferAndSeller(productOffer.getPrice(), seller.map(Seller::getSellerName).orElse(null))))
                            .collect(Collectors.toList());
                    CompletableFuture<Void> allDoneFuture = CompletableFuture.allOf(productOfferAndSellerFutures.toArray(new CompletableFuture[0]));
                    return allDoneFuture.thenApply(v -> productOfferAndSellerFutures.stream()
                            .map(CompletableFuture::join).
                                    collect(Collectors.toList()));
                 });
      } else {
         return CompletableFuture.supplyAsync(() -> offerService.getProductOffers(productId).stream()
                 .map(productOffer -> {
                    final Optional<Seller> seller = sellerService.getSeller(productOffer.getSellerId());
                    return new ProductOfferAndSeller(productOffer.getPrice(), seller.map(Seller::getSellerName).orElse(null));
                 })
                 .collect(Collectors.toList()));
      }
   }

   private CompletableFuture<Pair<List<String>, Double>> getProductReviewsReroute(String productId) {
      if (useKotlin) {
         return reviewServiceKt.getProductReviewsForJavaCall(productId)
                 .thenApply(this::getReviewsAndRating);
      } else {
         return CompletableFuture.supplyAsync(() -> getReviewsAndRating(reviewService.getReviews(productId)));
      }
   }

   @NotNull
   private Pair<List<String>, Double> getReviewsAndRating(List<ProductReview> reviews) {
      final List<String> allReviews = new ArrayList<>();
      double rating = 0.0;
      for (ProductReview review : reviews) {
         allReviews.add(review.getReviewNote());
         rating += review.getStar();
      }
      if (reviews.size() != 0) rating = rating / reviews.size();
      return Pair.of(allReviews, rating);
   }
}
```

The code above shows the complexity working with `CompletableFuture`. If you take a look at `getProductOfferAndSellersReroute` method,
then you will see an example of composing `CompletebleFuture` pipeline. [Spring Reactor](https://projectreactor.io/) has slightly better operator naming. However, 
there are tons of operators that we have to learn to use it properly.

Beside `AggregatorService`, we need to adjust unit tests in [`AggregatorServiceTests`](https://github.com/dpranantha/kotlin-coroutine-interops/blob/feature/migrate-all-calls-except-aggregator/src/test/java/com/dpranantha/coroutineinterops/service/AggregatorServiceTests.java) 
and [`AggregatorServiceKtTests`](https://github.com/dpranantha/kotlin-coroutine-interops/blob/feature/migrate-all-calls-except-aggregator/src/test/kotlin/com/dpranantha/coroutineinterops/service/AggregatorServiceKtTests.kt). The changes in unit tests 
are simple, i.e., adding mocks for both `AggregatorServiceTests` and `AggregatorServiceKtTests`, 
and changing service calls to Kotlin version in `AggregatorServiceKtTests` (along with making the mocks to return `CompletableFuture`). 
This make sures that the functionality of aggregation logic remains the same for both Java and Kotlin.

We also need to adjust the expected execution time in the integration tests into a lower value as we're using `CompletableFuture`
for both Java and Kotlin cases. In the previous post, we mentioned that the original Java version has around 1200 ms processing time. 
Migrating `ProductDescriptionService` decreased the processing time by 200 ms, i.e. around 1000 ms. After migrating the all service calls,
we managed to decrease the processing time further by 400 ms. 

![Expected processing time java](/assets/images/kotlin-coroutine-2/execution-time-all-calls-are-completable-future.png "Expected processing time after migrating all service calls")

The next step is to migrate `AggregationService` to Kotlin. This means we adopt the `AggregationService` logic into `AggregationServiceKt`
and start using `AggregationServiceKt` in the `AggregatorController` controller via `useKotlin` flag and in the `AggregationServiceKtTests` unit tests.
Changes in both [`AggregatorController`](https://github.com/dpranantha/kotlin-coroutine-interops/blob/feature/migrate-aggregator-service-use-in-controller/src/main/java/com/dpranantha/coroutineinterops/controller/AggregatorController.java) 
and [`AggregationServiceKtTests`](https://github.com/dpranantha/kotlin-coroutine-interops/blob/feature/migrate-aggregator-service-use-in-controller/src/test/kotlin/com/dpranantha/coroutineinterops/service/AggregatorServiceKtTests.kt) are straightforward.

```kotlin
@Service
class AggregatorServiceKt(
    private val productCatalogServiceKt: ProductCatalogServiceKt,
    private val productDescriptionServiceKt: ProductDescriptionServiceKt,
    private val productOfferServiceKt: ProductOfferServiceKt,
    private val sellerServiceKt: SellerServiceKt,
    private val productReviewServiceKt: ProductReviewServiceKt
) {

    @Throws(ProductNotFoundException::class)
    fun getProductSummaryForJavaCall(productId: String): CompletableFuture<ProductSummary> = GlobalScope.future { getProductSummary(productId) }

    @Throws(ProductNotFoundException::class)
    suspend fun getProductSummary(productId: String): ProductSummary = coroutineScope {
        val productCatalogDeferred = async { productCatalogServiceKt.getProductCatalog(productId) }
        val productCatalog = try {
            productCatalogDeferred.await()
        } catch (e: Exception) {
            logger.error("Error retrieving data for product catalog with productId {}: {}", productId, e.cause?.localizedMessage)
            null
        }
        if (productCatalog == null) throw ProductNotFoundException("Product can't be found!") //simplicity, throw Product Not Found
        else
            supervisorScope {
                val productDescriptionDeferred = async { productDescriptionServiceKt.getProductDescription(productId) }
                val productOffersDeferred = async { productOfferServiceKt.getProductOffers(productId) }
                val productReviewsDeferred = async { productReviewServiceKt.getProductReviews(productId) }

                val productDescription = try{
                    productDescriptionDeferred.await()
                } catch (e: Exception) {
                    logger.error("Error retrieving data for product description with productId {}: {}", productId, e.cause?.localizedMessage)
                    null
                }

                val productReviews = try {
                    getReviewsAndRating(productReviewsDeferred.await())
                } catch (e: Exception) {
                    logger.error("Error retrieving data for product review with productId {}: {}", productId, e.cause?.localizedMessage)
                    Pair.of(emptyList(), 0.0)
                }

                val productOfferAndSellers = try {
                    productOffersDeferred.await()
                        .map {
                           // Note: this is a sequential map, hence it is kind of useless to have coroutine implementation in here
                            supervisorScope {
                                val sellerDeferred = async(Dispatchers.IO) { sellerServiceKt.getSeller(it.sellerId) }
                                try {
                                    val seller = sellerDeferred.await()
                                    ProductOfferAndSeller(it.price, seller?.sellerName)
                                } catch (e: Exception) {
                                    logger.error("Error retrieving data for seller with sellerId {}: {}", it.sellerId, e.cause?.localizedMessage)
                                    ProductOfferAndSeller(it.price, null)
                                }
                            }
                        }
                        .toList()
                } catch (e: Exception) {
                    logger.error("Error retrieving data for product offer and seller with productId {}: {}", productId, e.cause?.localizedMessage)
                    emptyList()
                }
                
                ProductSummary(
                    productId,
                    productCatalog.productName,
                    productDescription?.shortDescription,
                    productDescription?.weightInKg,
                    productDescription?.color,
                    productOfferAndSellers,
                    productReviews.left,
                    productReviews.right)
            }
    }

    private fun getReviewsAndRating(reviews: List<ProductReview>): Pair<List<String>, Double> {
        val allReviews: MutableList<String> = ArrayList()
        var rating = 0.0
        for (review in reviews) {
            allReviews.add(review.reviewNote)
            rating += review.star.toDouble()
        }
        if (reviews.isNotEmpty()) rating /= reviews.size
        return Pair.of(allReviews, rating)
    }

    companion object {
        private val logger = LoggerFactory.getLogger(AggregatorServiceKt::class.java)
    }
}
```

In general, concurrent programming is hard. The Kotlin coroutine version is not trivial. The `CompletableFuture` version 
has a better execution time than the Kotlin coroutine, i.e., 600-650 ms vs 800-900 ms. One reason is that in Kotlin coroutine, 
it has a sequential map operation which calls `SellerServiceKt.`

![Expected processing time kotlin](/assets/images/kotlin-coroutine-2/execution-time-all-calls-are-coroutine-sequential-map.png "Expected processing time after migrating all service calls and aggregate service into using Kotlin coroutine")

Making the map operation concurrent (or parallel) is a bit involved. Some ideas: 
1. Transform the Kotlin list into Java parallel stream. However, this means we operate within the Java realm, and we lose the ability to use Kotlin coroutine.
2. Transform the Kotlin list into [Kotlin flow](https://kotlinlang.org/docs/flow.html). The problem is 
   that a map operation in Kotlin flow is also sequential. We can use fusing operations such as `channelFlow`, `buffer`, etc. 
3. Perform a concurrent loop. It requires us to join two lists: product offers list and sellers list into a single product offer and sellers. 
   It is not a very fluent one.

At this time of writing, we can try the suggestion in [kotlinx open issue](https://github.com/Kotlin/kotlinx.coroutines/issues/1147). 

```kotlin
val productOfferAndSellers = try {
    productOffersDeferred.await()
        .asFlow()
        .concurrentMap(this, 5) { //concurrent map
            supervisorScope {
                val sellerDeferred = async(Dispatchers.IO) { sellerServiceKt.getSeller(it.sellerId) }
                try {
                    val seller = sellerDeferred.await()
                    ProductOfferAndSeller(it.price, seller?.sellerName)
                } catch (e: Exception) {
                    logger.error("Error retrieving data for seller with sellerId {}: {}", it.sellerId, e.cause?.localizedMessage)
                    ProductOfferAndSeller(it.price, null)
                }
            }
        }
        .toList()
} catch (e: Exception) {
    logger.error("Error retrieving data for product offer and seller with productId {}: {}", productId, e.cause?.localizedMessage)
    emptyList()
}
```

The `concurrentMap` method is available below.

```kotlin
private fun <T, R> Flow<T>.concurrentMap(scope: CoroutineScope, concurrencyLevel: Int, transform: suspend (T) -> R): Flow<R> = this
        .map { scope.async { transform(it) } }
        .buffer(concurrencyLevel)
        .map { it.await() }
```

The approach adds one more `async` coroutine level and uses a buffer to execute a flow map operation concurrently. It reduces 
the execution time by 200 ms.

![Expected processing time kotlin - 2](/assets/images/kotlin-coroutine-2/execution-time-all-calls-are-coroutine-concurrent-map.png "Expected processing time with concurrent flow map")

We still keep our objectives:
1. Existing unit tests should remain valid with no/little adjustment.
2. Additional unit tests are created to check concurrency.
3. Integration tests should remain valid.

A conclusion is that composing Kotlin coroutines are challenging. Using Kotlin flow can help with its useful operator,
albeit not as extensive as reactive streams implementation such as [rxJava](https://github.com/ReactiveX/RxJava) 
and [Spring Reactor](https://projectreactor.io/).

In the next part, we will discuss moving `AggregateController` into Kotlin codebase, removing all Java codes, 
and using the event-loop-based server by going completely reactive.

Have fun coding!

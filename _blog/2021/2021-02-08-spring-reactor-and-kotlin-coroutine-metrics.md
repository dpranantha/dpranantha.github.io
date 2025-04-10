---
layout: post
title: "Measuring execution time in Spring Reactor Mono and Kotlin coroutine"
permalink: /measuring-exec-time-mono-and-kotlin-coroutine
date: 2021-02-09 21:30:00+01:00
categories:
- Coding
tags:
- Coroutine
- Thread
- Metrics
- Micrometer
- Prometheus
collection: blog
---

I use Kotlin coroutine and Spring Reactor extensively for back-end development. 
Some challenges that I've encountered using those libraries are measuring execution time, 
recording latency buckets, and publishing statistical metrics.

Statistical metrics such as histograms and summaries are complex. It is useful to calculate percentiles (90p, 99p, etc.) 
or the percentage of requests that fall under a time-bucket criterion (e.g., less than 200 ms).
It is in turn used to establish latency SLI (Service Level Indicator) and set up latency SLO (Service Level Objective) 
based on [Google SRE practices](https://cloud.google.com/blog/products/gcp/sre-fundamentals-slis-slas-and-slos). 
Besides, we can approximate [Apdex score](https://en.wikipedia.org/wiki/Apdex) based on the gathered metrics.

The additional challenge is that Kotlin coroutine and Spring Reactor are asynchronous (and concurrency) libraries. 
Thus, we cannot measure things simply using AOP (aspect oriented programming) annotations. 

This writing explains one approach in creating statistical metrics for both Kotlin coroutine and Spring Reactor using codes in Kotlin. 
A [Spring Boot](https://spring.io/projects/spring-boot) REST-API service and several unit tests demonstrates the usage and the results.
[Micrometer Prometheus](https://micrometer.io/docs/registry/prometheus) records the application metrics. The diagram
below describes the structure of the application.

![Xkcd Rest Api Architecture](/assets/images/xkcd-service/async-metrics.png "Xkcd Rest Api architecture")

The system has three main components: 
1. `XkcdClient` to perform external REST API call to [Xkcd](https://xkcd.com/) using [Spring Webclient](https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/html/boot-features-webclient.html) 
   and [Resilience4J](https://resilience4j.readme.io/).
2. `XkcdService` to map the response into `Service X` format. 
3. `XkcdController` to provide REST API controller implementation.

Each component has Spring Reactor with mono and Kotlin coroutine versions. Our goal is to measure each
component. We expect that the execution time to be `XkcdController` >= `XkcdService` >= `XkcdClient`. Let's take a look
at each version.

### 1. Measuring Kotlin coroutine
To demonstrate our case, let's annotate the suspend function in `XkcdService` with `@Timed` 
while the rests use the measurement utility (I'll come back to the utility right after that). 
We then configure both `TimeAspect` and `Async` [configuration](https://micrometer.io/docs/concepts#_the_timed_annotation) 
since the setup is applicable to Java `CompletableFuture`. The codebase for `XkcdService` looks like below.

```kotlin
@Service
class XkcdService(private val xkcdClient: XkcdClient) {
  
    @Async //**DON'T DO THIS!!!** - this will blow up your application
    @Timed("service.getComicById")
    suspend fun getComicById(id: String): Xkcd? = coroutineScope {
        xkcdClient
            .getComicById(id)
            ?.let { Xkcd(it.num, it.img, it.title, it.month, it.year, it.transcript) }
    }

    //Mono version is truncated for brevity
}
```

This setup won't work (at the time of writing). This what will happen to the execution:
1. Spring `threadPoolTaskExecutor` will execute `suspend fun getComicById(id: String): Xkcd?`.
2. Given `xkcdClient.getComicById(id)` is **a suspendable method**, if `xkcdClient.getComicById(id)` method uses the same context, i.e., `coroutineScope`, 
   then the `threadPoolTaskExecutor` will also execute the method. Otherwise, a thread from the corresponding Dispatcher threadpool will execute the method.
3. Either case, this will fail due to `kotlinx.coroutines.JobCancellationException: Parent job is Cancelled` due to mixed uses of frameworks.
   The parent aborts itself immediately after execution.
4. Worst case is it gets `kotlinx.coroutines.CoroutinesInternalError: Fatal exception in coroutines machinery for DispatchedContinuation`. 
   In this case, the coroutine tries to continue with `Dispatcher.Unconfined` and fails to get the context.
   
What if we get rid of the async stuff? 

```kotlin
@Service
class XkcdService(private val xkcdClient: XkcdClient) {
  
    @Timed("service.getComicById")
    suspend fun getComicById(id: String): Xkcd? = coroutineScope {
        xkcdClient
            .getComicById(id)
            ?.let { Xkcd(it.num, it.img, it.title, it.month, it.year, it.transcript) }
    }

    //Mono version is truncated for brevity
}
```

This what will happen to the execution:
1. The REST API works and returns a proper 200 response in Swagger.
2. The metrics are inaccurate. Instead of obtaining `XkcdController` >= `XkcdService` >= `XkcdClient`, it gets 
   - `XkcdController` (**1023 ms**) >= `XkcdService` (**11 ms**) <= `XkcdClient` (**960 ms**)
3. This happens because `@Timed` measures execution **until the method reaches suspension point**.

At this point, it is time to visit the measurement utility for Kotlin's coroutine I mentioned before. 
The idea is pretty straightforward. If we encapsulate a `Timer` in a suspendable method, then 
the suspended method should carry a timer state. Thus, the timer will continue measuring execution 
after recovering from a suspension point. Then, we can extract this pattern into a `suspend fun` 
that invokes to be measured `suspend fun`.

> **Idea**:
>  - Wrap a suspendable method execution inside another suspendable method with a timer in it.

We can implement any complex measurement using that approach, such as percentiles and time buckets. We can also add labels and taggings
as needed. The code below shows one possible implementation for the idea **minus error handling**. Unit tests are also available at the [Github repo](https://github.com/dpranantha/async-metrics).

```kotlin
suspend fun <T: Any> coroutineMetrics(
    suspendFunc: suspend () -> T,
    functionName: String,
    moreTags: Map<String, String> = emptyMap(),
    timeBuckets: Array<Duration> = DEFAULT_TIME_BUCKETS,
    meterRegistry: MeterRegistry
): T = coroutineMetricsWithNullable(suspendFunc, functionName, moreTags, timeBuckets, meterRegistry)!!

suspend fun <T: Any> coroutineMetricsWithNullable(
    suspendFunc: suspend () -> T?,
    functionName: String,
    moreTags: Map<String, String> = emptyMap(),
    timeBuckets: Array<Duration> = DEFAULT_TIME_BUCKETS,
    meterRegistry: MeterRegistry
): T? {
    require(timeBuckets.isNotEmpty()) { "timeBuckets are mandatory to create latency distribution histogram" }
    val timer = statisticTimerBuilder(
        metricsLabelTag = functionName,
        moreTags = moreTags,
        timeBuckets = timeBuckets
    )
        .register(meterRegistry)
    val clock = meterRegistry.config().clock()
    val start = clock.monotonicTime()
    try {
        return suspendFunc.invoke()
    } finally {
        val end = clock.monotonicTime()
        timer.record(end - start, TimeUnit.NANOSECONDS)
    }
}
```

We can then use the function as follows.

```kotlin
@Service
class XkcdService(
    private val xkcdClient: XkcdClient,
    private val meterRegistry: MeterRegistry
) {

    suspend fun getComicById(id: String): Xkcd? = coroutineScope {
        coroutineMetricsWithNullable(
            suspendFunc = suspend {
                logger.debug("Thread XkcdClient: ${Thread.currentThread().name}")
                xkcdClient
                    .getComicById(id)
                    ?.let { Xkcd(it.num, it.img, it.title, it.month, it.year, it.transcript) }
            },
            functionName = "service.getComicById",
            meterRegistry = meterRegistry
        )
    }

    //Mono version is truncated for brevity
}
```

This what will happen to the execution:
1. The REST API works and returns a proper 200 response in Swagger.
2. The metrics are accurate. It gets
    - `XkcdController` (**1003 ms**) >= `XkcdService` (**1000 ms**) >= `XkcdClient` (**942 ms**)
3. This means that the timer continues after the coroutine resumed from a suspension point and the approach works.

Despite the working approach, we still have to consider [structured concurrency](https://kotlinlang.org/docs/reference/coroutines/basics.html#structured-concurrency), e.g. 
the use of `supervisorScope`. I leave this as an exercise to the readers on handling it, as it differs per use-case.

> **Disclaimer**: we still need to think about structured concurrency per use-case and check if the approach works or needs to be adjusted.

### 2. Measuring Spring Reactor mono
The idea for measuring execution in Spring Reactor `mono` is similar to Kotlin coroutine by inserting a `Timer` via operation chaining. 
It is simpler compared to coroutine but with caveats on its own. Some of them include multiple channels in the case of `complete`, `error`, and `canceled`, if the mono
is empty, etc.

> **Idea**:
> - Transform mono into measurable mono with a `Timer` via operation chaining

The code below shows one possible implementation for the idea. Unit tests are also available at the [Github repo](https://github.com/dpranantha/async-metrics).

```kotlin
fun <T> Mono<T?>.withStatisticalMetrics(
        flowName: String,
        moreTags: Map<String, String> = emptyMap(),
        timeBuckets: Array<Duration> = DEFAULT_TIME_BUCKETS,
        meterRegistry: MeterRegistry
): Mono<T?> {
    require(timeBuckets.isNotEmpty()) { "timeBuckets are mandatory to create latency distribution histogram" }
    return this.name(flowName)
        .metrics()
        .assembleWithStatistics(flowName, moreTags, timeBuckets, meterRegistry)
}

private fun <T> Mono<T?>.assembleWithStatistics(
    metricsLabelTag: String,
    moreTags: Map<String, String>,
    timeBuckets: Array<Duration>,
    meterRegistry: MeterRegistry
): Mono<T?> {
    var subscribeToTerminateSample: Timer.Sample? = null

    val subscribeToCompleteTimer = statisticTimerBuilder(
        metricsLabelTag,
        "complete",
        moreTags,
        timeBuckets
    )
        .register(meterRegistry)

    val subscribeToCancelTimer = statisticTimerBuilder(
        metricsLabelTag,
        "cancel",
        moreTags,
        timeBuckets
    )
        .register(meterRegistry)

    val subscribeToErrorTimer = statisticTimerBuilder(
        metricsLabelTag,
        "error",
        moreTags,
        timeBuckets
    )
        .register(meterRegistry)

    return this
        .doOnSubscribe { subscribeToTerminateSample = Timer.start(meterRegistry) }
        .doOnSuccess { subscribeToTerminateSample?.stop(subscribeToCompleteTimer) }
        .doOnCancel { subscribeToTerminateSample?.stop(subscribeToCancelTimer) }
        .doOnError { subscribeToTerminateSample?.stop(subscribeToErrorTimer) }

}
```

We can then use the `mono` extension function as follows.

```kotlin
@Service
class XkcdService(
    private val xkcdClient: XkcdClient,
    private val meterRegistry: MeterRegistry
) {

    //suspend function is truncated for brevity
    
    fun getComicByIdAsMono(id: String): Mono<Xkcd?> = Mono.defer {
        logger.debug("Thread XkcdClient: ${Thread.currentThread().name}")
        xkcdClient
            .getComicByIdAsMono(id)
            .map { it?.let {
                Xkcd(it.num, it.img, it.title, it.month, it.year, it.transcript) } }
    }.withStatisticalMetrics(
        flowName = "service.getComicByIdAsMono",
        meterRegistry = meterRegistry
    )

    //logging is truncated  for brevity
}
```

This what will happen to the execution:
1. The REST API works and returns a proper 200 response in Swagger.
2. The metrics are accurate. It gets
    - `XkcdController` (**1370 ms**) >= `XkcdService` (**1343 ms**) >= `XkcdClient` (**1335 ms**)

> **Disclaimer**: we still need to think empty mono if needed. Unit tests provide an example for this.

### 3. Raw metrics
This section shows some raw measurements from the example. 

#### Coroutine raw measurement for quantile
```
service_getComicById_statistic_seconds{service="service.getComicById",quantile="0.5",} 0.142606336
service_getComicById_statistic_seconds{service="service.getComicById",quantile="0.9",} 0.142606336
service_getComicById_statistic_seconds{service="service.getComicById",quantile="0.99",} 0.142606336
```

#### Mono raw measurement for quantile 

```
service_getComicByIdAsMono_statistic_seconds{service="service.getComicByIdAsMono",status="complete",quantile="0.5",} 1.34217728
service_getComicByIdAsMono_statistic_seconds{service="service.getComicByIdAsMono",status="complete",quantile="0.9",} 1.34217728
service_getComicByIdAsMono_statistic_seconds{service="service.getComicByIdAsMono",status="complete",quantile="0.99",} 1.34217728
```

The only difference between both is that we have a `status` tag as mono works with different channels for handling success, error, and cancel.
Those are default tags and can be extended as needed.

### 4. Extra example: Spring Webclient extension for coroutine and mono
The example is available in the [Github repo](https://github.com/dpranantha/async-metrics) and used in the `XkcdClient` implementation. I will discuss the interesting part
which is mono and coroutine interoperability in another blog post.

### 5. Conclusion
- The example demonstrated how we measure asynchronous execution in Kotlin coroutine and Spring Reactor mono.
- To perform such task, we followed the pattern of the libraries in general. 
- Beware of mixing executors and dispatchers for Kotlin coroutine. Do this when you know what it entails.
- `@Timer` and `@Async` are suitable for Java `CompletableFuture`. Alternatively, we can transform a `suspend fun` 
  as a common `fun` returning `CompletableFuture` via [Kotlin coroutine jdk8 library]({{ site.baseurl }}/blog/2021/2021-01-18-kotlin-coroutines) 
  if we use Kotlin coroutine.

Have fun coding!

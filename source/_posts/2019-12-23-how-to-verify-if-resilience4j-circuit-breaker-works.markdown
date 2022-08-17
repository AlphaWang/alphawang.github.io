---
layout: post
title: "How to Verify If Resilience4j Circuit Breaker Works"
date: 2019-12-23 17:20:44 +0800
comments: true
categories: [distributed system]
tags: [distributed system, Resilience]
keywords: distributed system, Resilience4j, Circuit Breaker
description: 如何验证 Resilience4j Circuit Breaker
---

Resilience4j is a widely-used library which inspired by Hystrix, it helps with building fault tolerance distributed systems. We are using its circuit breakder module to prevent a cascade of failures when a remote service is down.

You can implement a circuit break based on [the official manual](https://resilience4j.readme.io/docs/examples), and you may want to verify if it works well or not after . Here're some tips

<!--more-->

## Circuit Breaker Configurations

Before the testing, it's a good idea to learn the Circuit Breaker Configurations, which are critical to a successful test. See some important configurations below:

- `failureRateThreshold`: When the failure rate is equal or greater than the threshold the CircuitBreaker transitions to open and starts short-circuiting calls.
- `minimumNumberOfCalls`: it means the minimum number of calls which are required (per sliding window period) before the CircuitBreaker can calculate the error rate. For example, if minimumNumberOfCalls is 10, then at least 10 calls must be recorded, before the failure rate can be calculated. If only 9 calls have been recorded the CircuitBreaker will not transition to open even if all 9 calls have failed.
- `slidingWindowSize` / `ringBufferSizeInClosedState (legacy)`: the size of the sliding window which is used to record the outcome of calls when the CircuitBreaker is closed.
- `recordExceptions`: A list of exceptions that are recorded as a failure and thus increase the failure rate. If you specify a list of exceptions, all other exceptions count as a success.

## Mock & Test

### Mock Thresholds

The thresholds in production enviroment are usually configured as a large value and not convenient for testing. For example, 

- `minimumNumberOfCalls` might be 100, so you have to make 100 requests to trigger the error rate calculation. 
- `slidingWindowSize / ringBufferSizeInClosedState` might be 100, so you have to make another 100 requests to complete the error rate calculation.
- `failureRateThreshold` might be 90, so you have to ensure 90% requests return exception so that the circuit breaker can be triggered to transit from CLOSED to OPEN.

In short, reducing those thresholds in testing environment is recommended. For example, here's how I set them:

- `minimumNumberOfCalls` = 1.
- `slidingWindowSize` = 5.
- `failureRateThreshold` = 10. (another option is mock a higher exception)

### Mock Exceptions

Next you need to mock exception when calling the remote service so that the circuit break can recognize the call as an error, then trigger the state transition after the error rate increased and reached the threshold. 

The first thing to notice is, not all exceptions will be counted in the error rate. So it is a good idea to figure out what kind of exception you need to mock first. If you've set `recordExceptions` on CircuitBreakerConfig, those exceptions are what you need to mock!

For example, below snippet means only ApiServerException will be counted in the error rate.

```
private static final CircuitBreakerConfig DEFAULT_CONFIG = CircuitBreakerConfig.custom()
		.failureRateThreshold(10)
		.slidingWindow(5, 1, SlidingWindowType.COUNT_BASED) 
		.recordExceptions(ApiServerException.class) 
		.build();
```

So we need to mock ApiServerException during the remote call.

```
private Object invoke(Object adapter, Object fluentRequest) {
		if (count.incrementAndGet() % 2 == 0) {
			throw new ApiServerException("MOCK-EXCEPTION");
		}
		
		return remoteService.call();
}
```

> Note: this example mocks the exception on api client side, you can also mock on the server side.

### Testing

Now you can request the target API, if the circuit breaker works well, you can find the state transit from CLOSED to OPEN once the error rate reachs the theshold. You may find a log like:

```
CircuitBreaker 'xx' changed state from CLOSED to OPEN
```

> Note: assumed you've attached onStateTransition event.
>
> ```
> circuitBreaker.getEventPublisher().onStateTransition(event ->
> 			log.warn("{}, {}", circuitBreaker.getName(), event.getStateTransition()));
> ```

## Conclusion

In short, reducing those thresholds in testing environment is recommended, and mock exception when calling the remote service, then you can verify the state transition by monitoring the logs.

## One More Thing

You may found changing the thresholds is a tedious job which need to change the code and deploy, I recommend store the thresholds in a [Configuration Center](https://medium.com/@theparanoidengr/an-overview-of-config-management-in-microservices-architecture-efa815035316), so that you can config a lower value for testing environment without touching the code while not affect production environment. 

Another benefit of integrating it with Configuration Center is that you can easily adjust the thresholds to fit the real world at any time. 








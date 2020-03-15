---
title: "Resilience in microservices era"
date: 2020-03-15T12:21:26+01:00
draft: false
categories: ["Development"]
tags: ["resilience4j", "java", "patterns", "circuit-breaker"]
---

How many times did you have to integrate an external service? Every day we say sentences like:

- this HTTP call takes too long
- this stuff breaks all the times

All these complaints are typical when a remote call should be performed, especially when we have to deal with unresponsive systems, which can cause a cascade effect that can be hard to come back from.

This is even more true when we have to work with a microservices architecture with synchronous HTTP calls.

# Resiliency

This kind of problems will always exist. However, there is a bunch of patterns and practices that can help mitigating their consequences, incrementing our services resilience.

## Circuit Breaker

The basic idea behind the circuit breaker is very simple. You wrap a protected function call in a circuit breaker object, which monitors for failures. Once the failures reach a certain threshold, the circuit breaker trips, and all further calls to the circuit breaker return with an error, without the protected call being made at all.

## Bulkhead

Bulkheads in ships separate components or sections of a ship such that if one portion of a ship is breached, flooding can be contained to that section.

The microservice bulkhead pattern is analogous to the bulkhead on a ship: failures in some component of a solution do not propagate to other components.

## Rete Limiter

Another really simple idea: if multiple calls to a remote service have to be made, you'd better split them during a certain range of time to make the external service able to serve all the requests.

# An example: resilience4J
Complete code is available [here](https://github.com/marcodenisi/resilience-microservices-era).

As stated by the official documentation, [resilience4j](https://github.com/resilience4j/resilience4j) is

> a lightweight, easy-to-use fault tolerance library inspired by Netflix Hystrix, but designed for Java 8 and functional programming.

Let's say we have a microservice exposing a `GET` API `/remote`, calling an external service:

{{<mermaid align="left">}}
sequenceDiagram

    participant U as User
    participant RC as ResilientController
    participant RS as ResilientService
    participant ES as ExternalService
    U ->> RC: GET /remote
    RC ->> RS: remoteCall()
    RS -->> ES: remoteCall()
    ES ->> ES: compute
    ES -->> RS: RemoteCallResult
    RS ->> RC: RemoteCallResult
    RC ->> U: RemoteCallResult
{{</mermaid>}}

Resilience4J allows us to decorate this call adding all the needed functionalities. Let's say we want to add a `CircuitBreaker`:

```java
@Service
public class ResilientService {

    @Autowired private ExternalService externalService;
    @Autowired private CircuitBreaker circuitBreaker;

    public RemoteCallResult getRemote() {
        return Decorators.ofSupplier(externalService::remoteCall)   // wraps the external call
                .withCircuitBreaker(circuitBreaker)                 // decorate with a circuit breaker
                .get();                                             // perform the call
    }
}
```

When we will call this method, we have 3 different scenarios, basing on the circuit breaker status:

- `CLOSED` status: external call is made
- `OPEN` status: external call is not made, an `CallNotPermittedException` is thrown
- `HALF_OPEN` status: only a limited amount of calls is made

The circuit breaker can obviously be configured as you wish, just check out the [official documentation](https://resilience4j.readme.io/docs/circuitbreaker) for all the details. In my example, I've written all the configuration needed in a *yaml* file:

```yaml
resilience4j.circuitbreaker:
    instances:
        remoteService:                                  // circuit breaker instance name
            registerHealthIndicator: true
            slidingWindowType: TIME_BASED               // time based check
            slidingWindowSize: 10                       // last 10 seconds
            permittedNumberOfCallsInHalfOpenState: 3
            minimumNumberOfCalls: 10                    // minimum number of calls to create a statistic
            waitDurationInOpenState: 30s
            failureRateThreshold: 50                
```

# Wrapping up

Introducing patterns like circuit breaker, bulkhead and others allow systems to be more resilient to failures, also putting in place recovering strategies.

Resilience4J is an extremely useful yet simple library to integrate and implement those patterns. Also, it's easy to add Grafana to have a better overall system monitoring.

<script async src="https://unpkg.com/mermaid@8.2.3/dist/mermaid.min.js"></script>
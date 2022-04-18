---
layout: post
title:  "Consumer Driven Contract Testing"
date:   2022-04-18 08:21:00 +0100
categories: microservices testing api
---

## Testing distributed systems

Modern architectures advocate increasingly for distributed systems. With distribution, you can scale independently not only 
the load of the system but also the team maintaining it.
However, this distribution comes with a cost: complexity. Distributed system are harder to operate, monitor ... and test.

In this scenario the number of integrations dramatically increases, and with it the need to test them.
The classical approach to test complex distributed system is the __end to end integration testing__. It works, but it's 
not easy to implement in real world, here are some of the reason that makes it difficult.

* __Dedicated test environment__: It requires a dedicated test environment with all deployed where to run the test.
* __Slow__: real requests to real systems with real integration, protocol overhead ...
* __Complex__: tests everything e2e at the same time. Then error might happen in a lot of different places, and for a 
lot of different reasons: environment setup, data inconsistency...
* __Coupled__: You need all the downstream systems working in order to test your integration.

## Consumers and providers

Besides, there are two different perspectives to each integration: The __provider__'s and the __consumer__'s.

Let's dig into the different concerns depending on the perspective. 

### Providers

![provider testing](/assets/img/2022-04-18-consumer-driven-contract-testing/provider-testing.svg)

The provider is the one exposing an interface, from a provider perspective, you want to verify the API behaves as expected.
There's a list of things you may want to assure:

* Input is properly parsed
* Business rules are applied
* Errors are properly handled
* Output format conforms the specifications
* Protocol metadata or any other protocol specific element is also used as expected


### Consumers

![consumer testing](/assets/img/2022-04-18-consumer-driven-contract-testing/consumer-testing.svg)

The consumer is the one using this API. The consumer perspective is way more complex to test. You want, for sure, a clear 
specification describing the message format, the error codes you might expect, available endpoints ...
But that's not enough, you'll need a deeper understanding on how the implementation works. Bug for bug compatibility.

## Mocks

Mocks are a good option for the consumers to test. The problem is: who builds the mock?

I there's no mock, the first option that comes to mind is, consumer carefully reads the specification and documentation
provided by the producer and builds the mock.

The problem here is that consumer will make assumptions, according their understanding, will require a lot of support on
the producer side, there might be misunderstandings and at the end _there's no way for the producer to test this mock
is faithful to the implementation_.

At this point is worth mention "The Law of Implicit Interfaces": If an interface has enough consumers, they will 
collectively depend on every aspect of the implementation, intentionally or not.
So from the producer side they also have to consider this when implementing changes. These changes must conform to both 
the explicitly documented interface, and the implicit interface captured by usage.
This is often referred as "bug-for-bug compatibility."

## Contract Testing

Contract testing is a methodology to address the problem previously exposed.
The idea is using the same contract test can be verified in both sides of the integration. This guarantees that there's
a proper understanding, not only of the documented spec but of the implementation details that conforms the implicit contract.

![consumer testing](/assets/img/2022-04-18-consumer-driven-contract-testing/contract-testing.svg)


There are two popular implementations:

* [Spring Cloud Contract](https://spring.io/projects/spring-cloud-contract)
* [Pact](https://docs.pact.io/)

### Spring Cloud Contract

Is more easy, and it's addressed to contract written in [Spring Boot](https://spring.io/projects/spring-boot), mainly REST
integrations.

It allows writing contracts in a simple DSL, and along the build process, it produces a so-called stub artifact which is
nothing but a [WireMock](https://wiremock.org/) stub.

This stub can be integrated by the consumers into their tests or can be run as a standalone process by teh WireMock 
stub runner.

Find here a sample test:

```javascript
contract {
    request {
        method = GET
        url = url("/v1/employees/000000")
        headers {
            accept = "application/json"
        }
    }
    response {
        status = NOT_FOUND
    }
}
```

And a [demo](https://github.com/scalvetr/contract-testing-demo) project showing the two sides, both the producer and the
consumer sides.

### Pact

On the other hand, **Pact** is a more complete solution, covering both the HTTP and message integrations. Another advantage
is that it is agnostic to the producer implementation technology.

**Consumer Driven Contracts**

The contract is generated during the execution of the automated consumer tests, and it is called **pact**. A **Pact Mock 
Provider** captures all the interactions happening within these tests. So, this means in this case the consumer side is 
always the one originating the contract.

Then a **Pact Mock Service** can be used to test the producer side.


![pact consumer testing](/assets/img/2022-04-18-consumer-driven-contract-testing/pact-contract-testing.png)

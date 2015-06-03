---
title: Acceptance testing service depending on Web API
author: wojciechkotlarski
---

Our current project is a first one where we have started using heavily an HTTP calls to communicate between our backend services.
Till then, all our solutions were based on asynchronous messaging, but we have decided to become more flexible in choosing a transport protocol that is fitting better to specific situation.
It means that for situations where we expect rather an immediate effect or response, we use synchronous HTTP calls, where for situations where we expect more that operation eventually happen but it does not have to happen immediately, we use asynchronous NServiceBus messaging.
When we started implementing this approach, we have realized that it has a big impact on our service level acceptance tests and the way how we perform them.

## How we perform acceptance tests

For each service that we are building, we have also a dedicated service level acceptance tests. They are executed every time when [CI](http://en.wikipedia.org/wiki/Continuous_integration) builds and deploy a new version of service after code change. After a successful code compilation and successful unit test execution, a service is being deployed on a test environment, and CI runs acceptance tests against it.
While all internal bits of service are wired up (i.e. its internal database, REST API endpoints and NServiceBus endpoint), all external dependencies are being mocked at this point.
It means that during those tests, service is not physically communicating to any other services, but instead our test framework is simulating their behaviour.
Testing service in isolation gives us benefits like:

* ability to test our service even if dependent component are not finished yet,
* better test stability, as our tests are not failing if dependent component is faulty at this moment,
* clear test output, as we know that a failed test means that our component has a bug, not like in system test, where a failed test can mean that our service, any of dependent services or infrastructure is a fauilure cause,
* better test performance, as all the operations are not going through entire system, but only through our components.

## Mocking dependencies that uses synchronous HTTP calls vs mocking ones using asynchronous messaging protocol

## Mocking Web API techniques
### Explicit API stubs
### MounteBank
### In process, adhoc mocks

## Cleaning state after each test
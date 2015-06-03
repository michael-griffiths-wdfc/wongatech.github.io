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
* clear test output, as we know that a failed test means that our component has a bug, not like in system test, where a failed test can mean that our service, any of dependent services or infrastructure is a failure cause,
* better test performance, as all the operations are not going through entire system, but only through our components.

## Mocking external dependencies

There are few options, how communication to external dependencies could be mocked.

### Mocking by changing implementation
The first option is that service code could contain a multiple implementations of classes responsible for communication.
For example, ```ICustomerDetailsProvider``` interface could have ```RestCustomerDetailsProvider``` and ```MockCustomerDetailsProvider``` implementation.
The service could have now a setting in app.config, specifying which implementation should be used on which environment, where in test environment a mock version would be used.

The advantage of this approach is simplicity, however there are few disadvantages as well:

* first of all, a different code is tested in comparison to the code that would be finally running on production,
* a production code may be potentially mixed with a test code (as ```MockCustomerDetailsProvider``` would be never used on production),
* any problems related to the implementation of ```RestCustomerDetailsProvider``` would be detected much later (during a system tests or even on production).

There are following examples of problems that ```RestCustomerDetailsProvider``` could cause, and which would not be detected early:

* data serialization issues (i.e. the used classes may not serialize properly because of some design constraints of used serialization framework),
* data mapping issues (i.e. domain model can be wrongly mapped to entities using during communication),
* error handling issues (i.e. a HTTP client can return different status codes than expected, causing different behaviours of service).

### Mocking by providing a fake version of external services
In this option, the mocking is not happening on a tested service side, but on a dependent service side.
While a service being under test uses a default communication mechanisms (i.e. makes an HTTP call or sends a NSB message), the dependent service is mocked.

This approach eliminates all the disadvantages of previous option, but the cost is that each of external service has to be somehow mocked.
Our team is using this approach, because proof that service is behaving correctly, and ability to detect and fix issues quickly has a bigger value than the cost of mocking services.

## Mocking dependencies basing on NServiceBus
Mocking NServiceBus services is a rather easy operation.
The service can communicate with NServiceBus target service in two ways:

* it can send messages (commands or events) to a target NSB service,
* it can handle messages (commands or events) coming from a target NSB service

The information, where the target NSB service is and how messages should be delivered to it, is located in app.config file and is presented below:

```xml
<UnicastBusConfig>
    <MessageEndpointMappings>
      <!-- Events that our service would be listening to-->
      <add Assembly="Wonga.FirstExternalDependency.Events"    Endpoint="first-dependency-queue-name@some-host" />

      <!-- Commands that our service would sending to external services-->
      <add Assembly="Wonga.SecondExternalDependency.Commadns" Endpoint="second-dependency-queue-name@some-host" />
    </MessageEndpointMappings>
  </UnicastBusConfig>
```

In order to isolate a service by mocking its dependencies, we have to redirect all those messages to a test queue on a host that runs tests, like:

```xml
<UnicastBusConfig>
    <MessageEndpointMappings>
      <!-- Events that our service would be listening to-->
      <add Assembly="Wonga.FirstExternalDependency.Events"    Endpoint="test-queue@test-env-host" />

      <!-- Commands that our service would sending to external services-->
      <add Assembly="Wonga.SecondExternalDependency.Commands" Endpoint="test-queue@test-env-host" />
    </MessageEndpointMappings>
  </UnicastBusConfig>
```

Now, during test execution, we can:

* assert that our service has sent a message (either command or event) to an external service, by analysing a test-queue content,
* emulate external service behavior by sending its messages (of both types) to our tested service queue.

Our test code doing that would look usually like below:
```c#
[Test]
private void Customer_should_receive_an_email_for_declined_application()
{
    Runner.RunScenario(
        given => An_application(),
        when  => Customer_submits_application(),
        and   => Decision_service_declines_application(),
        then  => Application_should_have_declined_status(),
        and   => An_email_with_decline_reason_should_be_sent_to_customer()
    );
}

private void An_application()
{
    _applicationId = CreateNewApplication();
}

private void Customer_submits_application()
{
    SubmitApplication(_applicationId);

    //Ensuring that RequestDecisionForApplication would appear in test-queue withing 3 seconds
    TestEndpoint.MessageReceiver.WaitFor<RequestDecisionForApplication>(
        TimeSpan.FromSeconds(3),
        request => request.ApplicationId == _applicationId,
        "The DecisionService was not triggered for ApplicationId={0}", _applicationId);
}

private void Decision_service_declines_application()
{
    //Simulating DecisionService decline
    TestEndpoint.Bus.Send<IApplicationDeclined>(message => message.ApplicationId = _applicationId);
}

private void Application_should_have_declined_status()
{
    // assert declined status on application
}

private void An_email_with_decline_reason_should_be_sent_to_customer()
{
    //Ensuring that SendDeclineEmail would appear in test-queue withing 3 seconds
    TestEndpoint.MessageReceiver.WaitFor<SendDeclineEmail>(
        TimeSpan.FromSeconds(3),
        request => request.ApplicationId == _applicationId,
        "The SendDeclineEmail was not sent to EmailService for ApplicationId={0}", _applicationId);
}
```

The good thing about mocking services basing on asynchronous call is that it is possible to model mocked service behaviour during test execution, so after initiating operation which would trigger communication with external service. 

?? should I include that?
The test can take a following form, which by the way is natural to the BDD scenarios:

1. Setup a data that would take a part in test (given)
2. Trigger an operation on service under test (when)
3. Ensure that service send a request to external service and respond with a response of the choice (when)
5. Verify that service reacted to it accordingly(then)
??

## Mocking dependencies basing on REST API

Mocking REST API (or any other HTTP based API) also looks simple. In this case, our service under test would have a setting like:
```xml
<appSettings>
    <add key="ExternalApiBaseUrl" value="http://some-host:1234/external-api" />
</appSettings>
```

When service is deployed in testing environment, it is a matter of changing this URL to point to a mock version of API.

What is more tricky is that unlike to NServiceBus mocks, HTTP calls are synchronous which means that an external service behaviour has to be mocked before test, not during it, because as soon as we initiate an operation on our service under test, it will attempt to call an external API and it will expect an immediate response.

?? should I include that
What it means is that the structure of test touching external API calls, has to be structured in a following way:

1. Setup a data that would take a part in test (given)
2. Setup all API mocks that would be called for given data to behave as expected (given?)
3. Trigger an operation on service under test (when)
4. Verify a result (then)
??

If we would imagine that scenario basing on NSB messages would use REST API, it will have to be restated into something like:

```c#
[Test]
private void Customer_should_receive_an_email_for_declined_application()
{
    Runner.RunScenario(
        given => An_application(),
        and   => Application_does_not_contain_all_required_details_to_be_accepted(),
        when  => Customer_submits_application(),
        then  => Application_should_have_declined_status(),
        and   => An_email_with_decline_reason_should_be_sent_to_customer()
    );
}
```

The step ```and => Application_does_not_contain_all_required_details_to_be_accepted()``` configures a mock API to ensure that application would be declined.

## Mocking Web API techniques
In above example, I have not shown the implementation of ```Application_does_not_contain_all_required_details_to_be_accepted()```. It is because it would be different, depending on which mocking technique we use.

### Explicit API stubs
When we started working with REST API, we were creating a stub version of all external APIs that our service was pointing to.
Those mocks were pretty static in a way, that it was always returning the same response, no matter of request content.
At that point we have not come to the point where different test scenarios would require a different mock service behaviour (like in example above, declining or accepting application).
Very quickly, we have realized that it is not a solution for us, as we knew that the service we are building is a kind of orchestration service and it will have multiple external dependencies.
If we would go with a dedicated API stub per external dependency, it would slow down our development a lot.
The reason for that was, that each stub API would require own project, own pipeline and it will have to be separately deployed on the testing environment boxes.

Later, during development, it will also limit our ability to test our service, because we will have to use a predefined, hard-coded identifiers to branch a behaviour of stub.

In other words, the implementation of a step configuring mock would be probably something like:
```c#
private void Application_does_not_contain_all_required_details_to_be_accepted()
{
    //a predefined application that would be declined by stub API
    _applicationId = Guid.Parse("857bfdcc-c6a2-4fb7-abfa-5d8eb081455d");
}
```

### Dynamically configurable generic stubs - MounteBank
Soon, when we realised that explicit API stubs are not the way we would like to go, we have found a [MounteBank](http://www.mbtest.org/) project, described in [Building Microservices](http://info.thoughtworks.com/building-microservices-book) book.

It looked a much more promising, because we could dynamically build a mock APIs.
We have created a simple wrapper api for it, to make it a bit easier to use from C# code, so we ended up with a implementation that for an above step would be something like:

```c#
private void Application_does_not_contain_all_required_details_to_be_accepted()
{
    var driver = new ApiStubDriver("http://test-env-host:2525/");

    driver.StubService(int.Parse(ConfigurationManager.AppSettings["DecisionApiPort"]), "http",
        new Stub(
            new StubRequest(HttpMethod.Get, string.Format("/get-decision/{0}", _applicationId)),
            new StubResponse(HttpStatusCode.OK)
                .WithBody("{\"decision\": \"declined\"}")
                .WithHeader("Content-Type","application/json")));
}
```

With MounteBank we could limit our external dependencies that we have to have in testing environment to one. It was a great improvement.

### In process, ad hoc mocks
While MounteBank was a great improvement for our tests, there were still a few drawbacks of using it:

* there was still physically a dependency of MounteBank itself, that needed to be deployed before we could run our tests (and we are running those tests in both: test environment, and locally on each developer box),
* the MounteBank instance became a shared component between all our test projects, so it created a risk that two projects running at the same time could interfere with each other, and it was very easily to achieve it by forgetting to change a PORT for one of stubbed API,
* finally the stubbed API definitions were being preserved between tests run as we have been not cleaning them up.

The overall effect was that the tests were not really executed in full isolation. While all of the problems stated above could be fixed and we could still use MounteBank happily, we have decided to use something more lightweight from deployment perspective.

Finally, we have switched to ad hoc mocks that are hosted in the same process where tests are being executed.
There is at least few open-source projects on GitHub that could be used for that:

* [HttpMock](https://github.com/hibri/HttpMock),
* [Moksy](https://github.com/greyham/Moksy),
* [SimpleHttpMock](https://github.com/xiaoyvr/SimpleHttpMock).

We are using the last one right now.

Finally, the example step configuring mock API with SimpleHttpMock would look like that:
```c#
private void Application_does_not_contain_all_required_details_to_be_accepted()
{
    var builder = new MockedHttpServerBuilder();

    builder.WhenGet(string.Format("/decision-api/get-decision/{0}", _applicationId))
        .RespondContent(HttpStatusCode.OK, r => new StringContent("{\"decision\": \"declined\"}", Encoding.UTF8, "application/json"));

    builder.Reconfigure(_mockApiServer, false);
}
```

Really, it is not much different in comparison to MounteBank. The difference is that here, the ```_mockApiServer``` is instantiated for each test and disposed after test finish, which means that each testing project and each test works on own, dedicated mock API, so they really run in isolation and there is no interference between tests.

In fact, there is one more change. While with MounteBank, we have been specifying an one fixed host with a dedicated PORT per mocked API, here we use the host where tests are being executed from (so the CI build agent one) and the same port for all of mocked APIs, but their URLs have prefixes (like /decision-api in this example). 

## Cleaning state after each test
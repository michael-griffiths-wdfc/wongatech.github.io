---
title: Acceptance testing service depending on Web API
author: wojciechkotlarski
---

Our current project is a first one where we have started using heavily HTTP calls to communicate between our services.
Till then, all our solutions were based on asynchronous messaging. We have decided however to become more flexible in choosing a transport protocol that is fitting better to specific situation.
It means that for situations where we expect rather an immediate effect or response, we use synchronous HTTP calls, where for situations where we expect more that operation eventually would finish but it does not have to happen immediately, we use asynchronous NServiceBus messaging.
When we started implementing this approach, we have realized that it has a big impact on our service level acceptance tests and the way how we perform them.

## 1. How we perform acceptance tests

For each service that we are building, we have a dedicated service level acceptance tests. They are executed every time when [CI](http://en.wikipedia.org/wiki/Continuous_integration) builds and deploys a new version of service. After a successful code compilation and successful unit test execution, the service is deployed on a test environment, and CI runs acceptance tests against it.
While all internal bits of service are wired up (i.e. its internal database, REST API endpoints and NServiceBus endpoint), all external dependencies are being mocked at this point.
It means that service is not physically communicating to any other services, but instead our test framework is simulating their behaviour.

Testing service in isolation gives us benefits like:

* ability to test our service even if dependent component are not finished yet,
* better test stability, as our tests are not failing if dependent component is faulty at given moment,
* clear test output, as we know that a failed test means that our component has a bug, not like in system tests, where our service, any of dependent services or infrastructure can be a cause of test failure,
* better test performance, as all the operations are not going through entire system, but only through our components.

## 2. Mocking external dependencies

There are few options, how communication to external services could be mocked.

#### Mocking by changing implementation
The first option is that service code could contain a multiple implementations of classes responsible for communication.
For example, ```ICustomerDetailsProvider``` interface could have ```RestCustomerDetailsProvider``` and ```MockCustomerDetailsProvider``` implementation.
The service could have now a setting in app.config, specifying which implementation should be used on which environment.
For test environment a mock version would be used then.

The advantage of this approach is simplicity, however there are few disadvantages as well:

* first of all, the tested code and code running on production environment is slightly different,
* a production code may be potentially mixed with a test code (as ```MockCustomerDetailsProvider``` implementation is always present, but never used on production),
* any problems related to the implementation of ```RestCustomerDetailsProvider``` would be detected much later (during a system tests or in worst scenario, after deployment to production).

There are following examples of problems that ```RestCustomerDetailsProvider``` could cause, and which would not be detected early:

* data serialization issues (i.e. classes used to transfer data may not serialize properly because due to some design constraints of serialization framework),
* data mapping issues (i.e. domain model can be wrongly mapped to entities used during communication),
* error handling issues (i.e. a HTTP client can return different status codes than expected, causing different behaviour of service).

#### Mocking by providing a fake version of external services

In this option, the mocking is not happening on a tested service side, but on a dependent service side.
While the service uses default communication mechanisms (i.e. makes an HTTP call or sends a NSB message), the target service is mocked.

This approach eliminates all the disadvantages of previous option, but the cost is that each of external service has to be mocked somehow.
Our team is using this approach, because proof that service behaves correctly, and ability to detect and fix issues quickly has a bigger value than the cost of mocking services.

## 3. Mocking dependencies that uses messaging protocol (NServiceBus)

Mocking NServiceBus services is a rather easy operation.
The service can communicate with NServiceBus target service in two ways:

* it can send messages (commands or events) to a target NSB service,
* it can handle messages (commands or events) coming from a target NSB service.

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

In order to mock service dependencies, we have to redirect all those messages to a test queue on a host that runs tests, like:

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

* assert that our service has sent a message to an external service, by reading messages from _test-queue_,
* emulate external service behavior by sending its messages to the service queue.

An example scenario and steps implementation could look as below:
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
}

private void Decision_service_declines_application()
{
    //Ensuring that RequestDecisionForApplication would appear in test-queue withing 3 seconds
    TestEndpoint.MessageReceiver.WaitFor<RequestDecisionForApplication>(
        TimeSpan.FromSeconds(3),
        request => request.ApplicationId == _applicationId,
        "The DecisionService was not triggered for ApplicationId={0}", _applicationId);

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

The good thing about mocking services that uses messaging is that it is possible to model mocked service behaviour during test execution, which gives more flexibility to model test scenarios.

## 4. Mocking dependencies that uses HTTP protocol (REST API)

Mocking REST API (or any other HTTP based API) also looks simple. In this case, our service under test would have a setting like:
```xml
<appSettings>
    <add key="ExternalApiBaseUrl" value="http://some-host:1234/external-api" />
</appSettings>
```

When service is deployed in testing environment, it is a matter of changing this URL to point to a mock version of API.

What is more tricky is that unlike to NServiceBus mocks, HTTP calls are synchronous which means that an external service behaviour has to be mocked before test, not during it.
It is because as soon as we initiate an operation on our service under test, it will attempt to call an external API, expecting an immediate response.

If the Decision Service from above example exposes REST API, the test scenario would have to be restated into something like:

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

The step ```and => Application_does_not_contain_all_required_details_to_be_accepted()``` configures a mock API to ensure that application would be declined when customer submits an application.

## 5. A various ways of mocking Web API dependencies

In above example, I have not shown the implementation of ```Application_does_not_contain_all_required_details_to_be_accepted()```.
It is because it would be different, depending on which mocking technique we use.

#### Explicit API stubs
When we started working on our project, we were creating a stub version of all external APIs that our service was pointing to.
Those mocks were pretty static in a way, that they were always returning the same response, no matter what request content was.
At that point we have not had a different test scenarios would require a different behaviour of mocked service (like declining or accepting application etc), however we knew that we would require such ability soon.

The other problem with this approach was that every new dependency required creation, deployment and maintenance of a dedicated API stubs.

Very quickly, we have realised that it is not a long term solution for us, as we knew that the service we are building is a kind of orchestration service and it will have multiple external dependencies.
If we would stick to this approach, it would slow down our development a lot.

If we would continue to follow this approach, we will have to use a predefined, hard-coded identifiers to branch a behaviour of stub, so the implementation of a step configuring mock would be probably something like:
```c#
private void Application_does_not_contain_all_required_details_to_be_accepted()
{
    //a predefined application that would be declined by stub API
    _applicationId = Guid.Parse("857bfdcc-c6a2-4fb7-abfa-5d8eb081455d");
}
```

#### Dynamically configurable generic stubs - MounteBank

Soon, when we realised that explicit API stubs are not the way we would like to go, we have found a [MounteBank](http://www.mbtest.org/) project, described in [Building Microservices](http://info.thoughtworks.com/building-microservices-book) book.

It looked much more promising, because we could dynamically build API stubs.
We have created a simple wrapper for it to make it a bit easier to use from C# code and we ended up with a implementation like:

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

With MounteBank we could limit our external dependencies that have to be present in testing environment to one. It was a great improvement.

#### In process, ad hoc mocks
While MounteBank was a great improvement for our tests, there were still a few drawbacks of using it:

* there was still a physical dependency on MounteBank itself, that had to be deployed before we could run our tests; we are running those tests on our test environment, and on each developer box, so the MounteBank service had to be pre-installed on all those boxes,
* the MounteBank instance became a shared component between all our test projects, so there was a risk that two projects running at the same time could interfere with each other,
* finally the stubbed API definitions were being preserved between tests run as we have been not cleaning them up, so those stubs could be used by other tests as well.

The overall effect was that the tests were not really executed in full isolation. While all of the problems stated above could be fixed and we could still use MounteBank happily, we have decided to use something more lightweight from a deployment perspective.

Finally, we have switched to ad hoc mocks that are hosted in the same process in which tests are being executed.
There is at least few open-source projects on GitHub that could be used for that:

* [HttpMock](https://github.com/hibri/HttpMock),
* [Moksy](https://github.com/greyham/Moksy),
* [SimpleHttpMock](https://github.com/xiaoyvr/SimpleHttpMock).

We are currently using the last one.

Finally, an example implementation with SimpleHttpMock could be like that:
```c#
private void Application_does_not_contain_all_required_details_to_be_accepted()
{
    var builder = new MockedHttpServerBuilder();

    builder.WhenGet(string.Format("/get-decision/{0}", _applicationId))
        .RespondContent(HttpStatusCode.OK, r => new StringContent("{\"decision\": \"declined\"}", Encoding.UTF8, "application/json"));

    builder.Reconfigure(_mockApiServer, false);
}
```

Really, it is very similar to the MounteBank. The difference is that, the ```_mockApiServer``` is instantiated for each test and disposed after test finish, which means that each test works on own API stubs.
All the tests are now running in isolation, so there is no interference between them.
Also, while with MounteBank, all the projects were using the same physical instance of MounteBank to stub API, here each build job that executes test project, uses own ad hoc stubs. 

## 6. Cleaning service state after each test

There is one more side effect of testing services that synchronously communicates with dependent services.

First, lets take a look back on the NSB version of scenario:
```c#
given => An_application(),
when  => Customer_submits_application(),
and   => Decision_service_declines_application(),
then  => Application_should_have_declined_status(),
and   => An_email_with_decline_reason_should_be_sent_to_customer()
```

Lets now imagine that when customer submits an application, the application itself changes status to 'submitted' state, and then decision request is being sent to the Decision Service.
What if we would like to just check, that the application status has been changed correctly?

With the NSB approach, we could just write scenario like:
```c#
given => An_application(),
when  => Customer_submits_application(),
then  => Application_should_have_submitted_status(),
```

It would be perfectly fine. Of course the service would still send a request to the Decision Service (like in original scenario), but as we are not going to do anything with it, the service will stop processing this application, because it would never receive a response from mocked Decision Service.

Now, if the same communication would be made via HTTP, the story would be slightly different.
Theoretically we still should be able to write test scenario in the same form, and if we run it, it should be green.
However, we will also see a lot of errors in the logs, as our service will not stop on changing application state to 'submitted', but it will be also attempting to retrieve decision from Decision Service.
Because we have not mocked this behaviour, the response would be invalid and operation would be failing, causing error entries appearing in logs. The service would probably try to retry the failing operation, causing even more errors. Those error entries would be blending easily with other errors signalling 'a real' issue with the service, so we were not happy about receiving them.

So, what we did to do to avoid such errors was to:

* mock all APIs with a default behaviour on test startup (where the API behaviour can be still customised during test),
* on test tear down, go through all applications that has been created during test and bring them to a final state, to ensure that service will not try to perform any more operations on them after test, so when mocked API definitions would not exist any more.

Those two changes allowed us to still have a clear test scenarios, without a need of adding steps just to ensure that tested service has finished work before test finish.
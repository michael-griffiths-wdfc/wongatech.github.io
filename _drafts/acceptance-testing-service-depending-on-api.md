---
title: Acceptance testing service depending on Web API
author: wojciechkotlarski
---

Our current project is the first one where we have started to make heavy use of HTTP communication between services.
Until now, our inter-service communication was almost exclusively based on asynchronous messaging. We have realized however that we need to introduce more flexibility in our architecture and that async is not always best.
It means that for situations when we expect an immediate effect or response, we use synchronous HTTP calls, and for situations where we are initiating a long running process, we use asynchronous NServiceBus messaging.
Once we started implementing this approach, we realized that it has a big impact on tests we run against our services and the how we execute them.

## 1. How we perform acceptance tests

For each service that we are building, we have dedicated service level acceptance tests. They are executed every time when our [Continuous Integration process](http://en.wikipedia.org/wiki/Continuous_integration) builds and deploys a new version of the service. After code is successfully compiled and unit tests execute successfully, the service is deployed on a test environment, and our CI environment triggers the acceptance tests against it.
While all internal parts of a service are wired up (i.e. its database, REST API endpoints and NServiceBus endpoint), all external dependencies are being mocked at this point.
It means that service is not physically communicating to any other services, but instead our test framework is simulating their behaviour.

![Acceptance testing][acceptance_tests.gif]

Testing a service in isolation gives us benefits like:

* The ability to test our service even if dependent components are not finished yet.
* Better test stability - our tests are not failing if dependent components are faulty at that time.
* Clear test output - we know a failed test means our component has a bug - not some other service.
* Better test performance - operations are not hitting the entire system, only through our components.

## 2. Mocking external dependencies

There are a few options when considering how to mock external services.

#### Mocking by changing the implementation
The first option is to replace the concrete implementation with a mocked implementation which mimics the behaviour of the external service.
For example, ```ICustomerDetailsProvider``` interface could have a concrete implementation ```RestCustomerDetailsProvider``` and a mock implementation ```MockCustomerDetailsProvider```.
The service can now specify in the applications configuration, which implementation should be used on which environment.
And in our test environment we can specify the mock implementation.

The advantage of this approach is simplicity, however there are few disadvantages as well:

* First of all, the tested code and code running on production environment is slightly different,
* There is a possibility that the mock implementation can get deployed in an environment where you want the real implementation since the Mock may be present in the code base,
* Any problems related to the implementation of ```RestCustomerDetailsProvider``` would be detected much later (during system tests or in the case worst scenario, after deployment to production).

The following are some examples of problems that won't be surfaced if the actual ```RestCustomerDetailsProvider``` implementation is not exercised:

* Data serialization issues - classes used to transfer data may not serialize properly.
* Data mapping issues - the domain model might be mapped incorrectly to  entities used for communication.
* Error handling issues - a HTTP client can return different status codes than expected, causing different behaviour of the service.

#### Mocking by providing a fake version of external services

With this option, the mocking does not happen within the service itself, but rather on the dependant service.  We simply configure our own service to talk to something that acts like the dependent service, this can be over any standard communication mechanism (e.g. HTTP or MSMQ).

This approach eliminates the disadvantages of the previous option, but the cost is having to manage and deploy a mocked version of the external service somehow.
Our team is using this approach, because of the strong guarantees it gives us that our service is working correctly, and the ability to detect and fix issues quickly has a bigger value than the extra cost of deploying mocked services.

## 3. Mocking dependencies that use a messaging protocol (NServiceBus)

Mocking the services which we communicate with over a message bus like NServiceBus is pretty straightforward.
The communication with another NServiceBus service can be grouped into two categories:

* **Outgoing** - We are sending messages (commands or events) to another NSB service,
* **Incoming** - We are handling messages (commands or events) coming from another NSB service.

The configuration detailing where the external NSB service is and how messages should be delivered to it, is located in our app.config file and looks something like this:

```xml
<UnicastBusConfig>
    <MessageEndpointMappings>
      <!-- Events that our service listens to-->
      <add Assembly="Wonga.FirstExternalDependency.Events"    Endpoint="first-dependency-queue-name@some-host" />

      <!-- Commands that our service sends to external services-->
      <add Assembly="Wonga.SecondExternalDependency.Commands" Endpoint="second-dependency-queue-name@some-other-host" />
    </MessageEndpointMappings>
  </UnicastBusConfig>
```

In order to mock service dependencies, we have to redirect all those messages to a test queue on a host that runs tests, like below (note the different endpoint configuration):

```xml
<UnicastBusConfig>
    <MessageEndpointMappings>
      <!-- Events that our service listens to-->
      <add Assembly="Wonga.FirstExternalDependency.Events"    Endpoint="test-queue@test-env-host" />

      <!-- Commands that our service sends to external services-->
      <add Assembly="Wonga.SecondExternalDependency.Commands" Endpoint="test-queue@test-env-host" />
    </MessageEndpointMappings>
  </UnicastBusConfig>
```

We achieve the desired change in the service configuration by applying a [CTT](https://ctt.codeplex.com/) transform when we deploy our service in the test environment (We actually perform this step for all environments to get the desired configuration not just in our test environments)

Now, during test execution, we can:

* Assert that our service has sent a message to an external service, by reading messages from _test-queue_,
* Emulate the behavior of an external service by sending its messages to the service queue.

An example scenario could look like this (The interesting stuff is in the ```Decision_service_declines_application``` method):
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
    //Ensure that our service sends the expected command to our external decisioning service (we timeout after 3 seconds if it does not happen)
    TestEndpoint.MessageReceiver.WaitFor<RequestDecisionForApplication>(
        TimeSpan.FromSeconds(3),
        request => request.ApplicationId == _applicationId,
        "The DecisionService was not triggered for ApplicationId={0}", _applicationId);

    //Simulating DecisionService decline, here we stuff the message we would expect to receive from Decisions into our own queue
    TestEndpoint.Bus.Send<IApplicationDeclined>(message => message.ApplicationId = _applicationId);
}

private void Application_should_have_declined_status()
{
    // assert declined status on application
}

private void An_email_with_decline_reason_should_be_sent_to_customer()
{
    //Ensure that our service sends the expected command to our external email service (we timeout after 3 seconds if it does not happen)
    TestEndpoint.MessageReceiver.WaitFor<SendDeclineEmail>(
        TimeSpan.FromSeconds(3),
        request => request.ApplicationId == _applicationId,
        "The SendDeclineEmail was not sent to EmailService for ApplicationId={0}", _applicationId);
}
```

The great thing about mocking services that use a message bus, is that it is possible to model mocked service behaviour during the test execution, which gives more flexibility to model test scenarios.

## 4. Mocking dependencies that uses HTTP protocol (REST API)

Mocking a REST API (or any other HTTP based API) also looks fairly simple. In this case, our service under test would have a setting like this:
```xml
<appSettings>
    <add key="ExternalApiBaseUrl" value="http://some-host:1234/external-api" />
</appSettings>
```

And when the service is deployed in testing environment, it is a matter of changing this URL to point to a mock version of API much like we did for the message bus example.

Now unlike mocks based on message bus communication, HTTP calls are synchronous which means that an external service behaviour has to be mocked before the test, not during it.
This is because once we initiate an operation on our service under test, it will attempt to call the external API, expecting an immediate response.  In contrast with asynchronous messaging for example where you can simply initiate the operation and stuff the message you expect as a response afterwards without any prior setup required.

If the Decision Service from the above example exposes REST API, the test scenario would have to be restated into something like:

```c#
[Test]
private void Customer_should_receive_an_email_for_declined_application()
{
    Runner.RunScenario(
        given => An_application(),
        and   => Application_does_not_contain_sufficient_data_to_be_accepted(),
        when  => Customer_submits_application(),
        then  => Application_should_have_declined_status(),
        and   => An_email_with_decline_reason_should_be_sent_to_customer()
    );
}
```

The step ```and => Application_does_not_contain_sufficient_data_to_be_accepted()``` configures a mock API to ensure that application would be declined when customer submits an application.  There is a few different ways this can be implemented which we will now explore.

## 5. Various ways of mocking Web API dependencies

#### Explicit API stubs
When we started working on our project, we were creating a stub version of all external APIs which were each deployed separatly.  We then configured our service to talk to these deployed mocks in our test environments.
Those mocks were pretty static, they were always returning the same response, no matter what request came in (e.g. for the Decision service we were always Accepting the application).
This worked reasonably well at the beginning as we didn't have a lot of different test scenarios requiring different behaviour of the mocked service (e.g. getting a declined application), however we knew that we would soon have that requirement and really the only way to achieve it with this setup would be to use masks (e.g. decline applications with a particular id) which didn't feel like a great idea and not very flexible.

In addition there is a reasonably large overhead in creating, deploying and maintaining what is effectively a whole new service for each and every external dependency you have just to provide mocking capabilities.

Very quickly, we realised that this was not going to work for us long term, as we knew that the service we were building was doing a bunch of orchestration and thus had a significant number external dependencies.  If we had to go and create/deploy/maintain a mock service for each one it was going to slow us down an awful lot.

#### Dynamically configurable generic stubs - MounteBank

After determining that the first approach wasn't suitable for us we did some research and came across a nice tool called [MounteBank](http://www.mbtest.org/), as described in Sam Newman's [Building Microservices](http://info.thoughtworks.com/building-microservices-book) book.

It looked much more promising, because we could dynamically build API stubs and reconfigure them on the fly which meant we did not have to create a whole new service for each external dependency, we could have just one Mountebank service deployed and during each test run we could configure it as we wish.

We created a simple wrapper for it to make it a bit easier to use from C# code and we ended up with a implementation like this:

```c#
private void Application_does_not_contain_sufficient_data_to_be_accepted()
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

With MounteBank we effectively reduced the external dependencies that have to be present in a testing environment to one which was much more manageable and it was a great improvement.

#### In process, ad hoc mocks
While MounteBank was a great improvement, there were still a few drawbacks with it:

* there was still a physical dependency on MounteBank itself, that had to be deployed before we could run our tests; we are running those tests on our test environment, and on each developer box, so the MounteBank service had to be pre-installed on all those boxes,
* the MounteBank instance became a shared component between all our test projects, so there was a risk that two projects running at the same time could interfere with each other,
* finally the stubbed API definitions were being preserved between tests run as we were not cleaning them up, so those stubs could unintentially impact other tests leading to flickering tests.

The overall effect of these problems is that the tests are not really executed in full isolation. While they can all be solved and you could use Mountebank very effectively for service tests, we were striving for something more lightweight from a deployment perspective.

Finally, we have switched to ad hoc mocks that are hosted within the same process in which tests are being executed.
There is at least few open-source projects on GitHub that could be used for that:

* [HttpMock](https://github.com/hibri/HttpMock),
* [Moksy](https://github.com/greyham/Moksy),
* [SimpleHttpMock](https://github.com/wongatech/SimpleHttpMock).

We are currently using the SimpleHttpMock.

Here is an example implementation of setting up a declined application using the ad-hoc SimpleHttpMock:
```c#
private void Application_does_not_contain_sufficient_data_to_be_accepted()
{
    var builder = new MockedHttpServerBuilder();

    builder.WhenGet(string.Format("/get-decision/{0}", _applicationId))
        .RespondContent(HttpStatusCode.OK, r => new StringContent("{\"decision\": \"declined\"}", Encoding.UTF8, "application/json"));

    builder.Reconfigure(_mockApiServer, false);
}
```

As you can see, it is very similar to MounteBank. The difference is that the ```_mockApiServer``` is instantiated for each test and disposed after each test finishes, which means that each test works on it's own API stubs and cannot affect another test giving much greater guarantees of test isolation.
Also, while with MounteBank, all the projects were using the same physical instance of MounteBank to stub the API, here each build job that executes test project, uses own ad hoc stubs, and because they are in the test runner's process they are guaranteed to be disposed of once the test runner finishes leading to a much cleaner environment.

## 6. Cleaning service state after each test

There is one more side effect of testing services that synchronously communicates with dependent services.

First, lets take a look back at the message bus version of the scenario:
```c#
given => An_application(),
when  => Customer_submits_application(),
and   => Decision_service_declines_application(),
then  => Application_should_have_declined_status(),
and   => An_email_with_decline_reason_should_be_sent_to_customer()
```

Let's now imagine that when a customer submits an application, the application itself changes status to 'submitted' state, and then the decision request is being sent to the Decision Service.
What if we would like to just check, that the application status has been changed correctly and don't care about any of the subsequent steps?

With the NSB approach, we could just write a scenario like this and it would be fine:
```c#
given => An_application(),
when  => Customer_submits_application(),
then  => Application_should_have_submitted_status(),
```

Of course the service would still send a request to the Decision Service (like in original scenario), but as we are not going to do anything with it, the service will stop processing this application, because it would never receive a response from mocked Decision Service.

Now, if the same communication would be made via HTTP, the story would be slightly different.
Theoretically we still should be able to write the test scenario in the same form, and if we run it, it should be green.
However, we will also see a lot of errors in the logs, as our service will not stop on changing application state to 'submitted', but it will be also attempting to retrieve a decision from the Decision Service.
Because we have not mocked this behaviour, the response would be invalid and operation would be failing, causing error entries in the logs. The service would probably attempt to retry the failing operation, causing even more errors. Those error entries would be blending easily with other errors signalling 'a real' issue with the service, so there would be a lot of noise and we would find it hard to diagnose the real problems.

So, what we did to avoid such errors was:

* mock all APIs with a default behaviour on test startup (where the API behaviour can still be customised during the test),
* on test tear down, go through all applications that have been created during the test and bring them to a stable final state, to ensure that service will not try to perform any more operations on them after our test, so when mocked API definitions would not exist any more.

Those two changes allowed us maintain simple test scenarios which have a clear intent, without adding steps just to ensure that tested service has finished work before the test finishes.

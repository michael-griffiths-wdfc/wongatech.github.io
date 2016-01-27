---
title: Turbocharge your development and testing with Angular Multimocks
author: nabilboag
---

If you develop web applications, you hopefully spend a lot of time testing that
your application works well. You can test your app manually by interacting with
the UI or you can write automated tests.

As a good engineer, you want to make sure your application works in a wide range
of different scenarios. Your customer is visiting for the first time, they're
making a purchase, they've requested a return, they're using a voucher code, and
so on. If your application makes API calls this means you should test your
application with different responses from your backend.

**You need to mock your backend.**

This post is going to explain the benefits of testing your application
with a mocked backend and how Angular Multimocks can help organise and
compose mocks for Angular applications.

## My app works already, why are you making me do more work?

If your goal in testing is to do a full integration test of your stack
then mocking the backend is probably counter-productive.

If your application makes very few API calls or the UX doesn't change
based on different API responses then Angular's built-in `ngMock` mocking
tools are great.

However, as applications grow in size and complexity, [integration tests
become increasingly
non-deterministic](http://martinfowler.com/articles/nonDeterminism.html).
Mocks allow you to control the test process carefully, but it can become
difficult to manage API response mocks, change scenarios and test complex
UX interactions.

## The hell that awaits those who do not mock

### The real world is asynchronous

When every API call is mocked during development, you don't experience what a
customer would when using your application. Developers might miss obvious
gaps in UX. E.g. forgetting to implement a loader animation when making a long
API call.

### Real backends are inconstant and inflexible

Instead of mocking an API some developers use a real backend. If you use a
real backend, ask yourself the following questions:

* What happens when my test fails?
* Was it my application or did the internet connection fail?
* Was the API team deploying code when I was running my test?
* What happens if the API is offline or has a version deployed that conflicts
  with my application?

One option is to use a fake backend service like [Mountebank](http://www.mbtest.org/).

### It's tough to get into your application's dark corners

So your backend team has built you an entire mock implementation of their
backend. This mock implementation will return standard responses for every
possible request. Great! You can run your tests.

But wait...

What if you want to test a non-standard scenario like error responses,
incomplete data or race conditions.

```javascript
// simple mock response using ngMock
$httpBackend.whenGET('/users').respond('foo');
```

What if I want the `GET` call to `/users` to return `bar` instead of `foo`?

## What about `ngMock`?

If your project has hard-coded mocks, then you know it can get messy.
Here's a really simple set of mocks for an Angular application:

```javascript
myAppDev = angular.module('myAppDev', ['myApp', 'ngMockE2E']);
myAppDev.run(function($httpBackend) {
  // some dummy data
  phones = [
    {name: 'phone1'},
    {name: 'phone2'}
  ];

  // return the current list of phones
  $httpBackend.whenGET('/phones').respond(phones);

  // add a new phone to the phones array
  $httpBackend.whenPOST('/phones').respond(function(method, url, data) {
    var phone = angular.fromJson(data);
    phones.push(phone);
    return [200, phone, {}];
  });

  // other stuff
  $httpBackend.whenGET('/users').respond('foo');
  // other stuff
  $httpBackend.whenGET('/cards').respond('bar');
  // other stuff
  $httpBackend.whenGET('/cart').respond('baz');
  // other stuff
  $httpBackend.whenGET('/foo').respond('fizzbuz');
  //... you get the point
});
```

As you add more responses and those responses grow in complexity, this
quickly becomes unmanageable.

## Enter our superhero: Angular Multimocks

Multimocks helps organise your mocks so that they are easier to read and more
maintainable.

- Create mock responses in JSON
- Create multiple scenarios with different responses and switch between them easily
- Add delays to mock responses to test asynchronous interactions
- Mocks can inherit from a default scenario to reduce duplication

With Multimocks, your mock responses are created in JSON and stored on the filesystem:

```json
{
  "httpMethod": "GET",
  "statusCode": 200,
  "uri": "/customer/cart",
  "response": {
    "id": "foo"
  }
}
```

With Multimocks, developers can have a more realistic UX by adding delays globally
or to individual API calls. This can be disabled during automated test execution to
keep run-time down.

### Scenarios

Multimocks allows you to change scenarios easily. It does this by allowing
you to compose “scenarios” out of different mock files using a manifest file.

Here's an example for an shopping cart app.

First we need a mock manifest to define the scenarios we want to test and which
responses are returned in each. The mock manifest below contains 2 scenarios, the
default (`_default`), with an empty shopping cart, and a second called `someItems`.


```json
{
  "_default": [
    "cart/empty.json"
  ],
  "someItems": [
    "cart/someItems.json"
  ]
}
```

Here is the default `empty.json` file, which contains a response for the URL
`/customer/123/cart`.

```json
{
  "httpMethod": "GET",
  "statusCode": 200,
  "uri": "/customer/123/cart",
  "response": {
    "items": []
  }
}
```

Here is the `someItems.json` file, which contains a response for the same URL,
but this time including one item in the response.

```json
{
  "httpMethod": "GET",
  "statusCode": 200,
  "uri": "/customer/123/cart",
  "response": {
    "items": [
      {
        "title": "REST in Practise",
        "type": "Book",
        "desc": "In this insightful book, three SOA experts provide...",
        "inStock": 4
      }
    ]
  }
}
```

When the user is in the default scenario a `GET` request to `/customer/123/cart`
will return the response defined in `empty.json`. When the user is in the
`someItems` scenario the response defined in `someItems.json` will be returned.

As you start to scale your application, the power of composable scenarios
will become apparent. The folder structure and the mock manifest in Angular
Multimocks makes it easy to simulate different states of your application.

### Another example

In another example, we might test an authentication feature. We might mock a
number of scenarios with different responses to the password authentication
API call. In the default scenario, the passcode authentication API call will
succeed. When the user is placed into the `passwordAuthFail` or
`passwordAuthLocked` scenarios, failure responses will be returned.

```json
{
  "_default": [
    "Account/_default.json",
    "Application/_default.json",
    "PasscodeAuthentication/success.json",
  ],
  "passwordAuthFail": [
    "PasswordAuthentication/fail.json"
  ],
  "passwordAuthLocked": [
    "PasswordAuthentication/locked.json"
  ]
}
```

## How does it work

Multimocks uses ngMock under the hood, which replaces Angular's `$httpBackend` service
(equivalent to monkey-patching `XMLHttpRequest` but less of a hack). The developer
creates mock files and then composes them into different scenarios.

## That is awesome, but what does it cost?

Mocks need to be maintained, if an API changes then the mocks need to be
updated to match. If not, your tests may pass and when deployed the app
might not work.

You will still need to do integration testing. At Wonga we also demo features to
stakeholders once they're done. As our stakeholders are familiar with the feature
requirements, they flag up issues and suggest more cases to test.

## Help make Multimocks better

We :heart: pull requests!

# Turbocharge your development and testing with Angular Multimocks

If you develop web applications, you hopefully spend a lot of time testing your
application works well. This can be done through end to end tests or manual
testing.

As a good engineer, you want to make sure your application works in **all**
possible situations. This means you should verify your application with
different responses from your backend.

**You need to mock your backend.**

This post is going to explain the importance of testing your application
with a mocked backend and how Angular Multimocks can help organise and
compose mocks.

## My app works already, why are you making me do more work?

If your application makes very few API calls or the UX doesn't change based
on different API responses then simple `$httpBackend` calls make sense.

As the scale of you application grows, it becomes difficult to manage the
mocks, change scenarios on the fly and test UX interactions.

## Cool, but do I really have to?

### Don’t fool yourself.

When every API call is mocked during development, you don't experience what a
customer would see when using your application.
Developers then don’t see what would be obvious gaps in UX. E.g. forgetting to
implement a loader GIF when making a long API call.

### Whoa, what is this mess?

If your project has hardcoded mocks, then you know it can get messy.

```javascript
myAppDev = angular.module('myAppDev', ['myApp', 'ngMockE2E']);
myAppDev.run(function($httpBackend) {
  phones = [{name: 'phone1'}, {name: 'phone2'}];

  // returns the current list of phones
  $httpBackend.whenGET('/phones').respond(phones);

  // adds a new phone to the phones array
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

### Don't use a real backend. No seriously, don't

Instead of mocking an API some developers use a real backend. If you use a
real backend, ask yourself the following questions:

* What happens when my test fails?
* Was it my application or did the internet connection fail?
* Was the API team deploying code when I was running my test?
* What happens if the API is offline or has a version deployed that conflicts with my application?

Some people get around these issues with a fake backend service that is
spawned locally.

### Change places!

So your backend team has built you an entire mock implementation of their
backend. This mock implementation will return standard responses for every
possible request. Great! You can run your tests.

But wait...

What if you want to test a non standard scenario like error responses,
incomplete data or race conditions.

```javascript
$httpBackend.whenGET('/users').respond('foo');
```

What if I want the `GET` call to `/users` to return `bar` instead of `foo`?

## Enter our superhero: Angular Multimocks

Multimocks helps organise your mocks so that they are easier to read and more
maintainable.

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

With Multimocks, developers can have a more realistic UX by adding delays to
API calls(or globally).

> This can be turned off during your tests so they don't take a million years.

Multimocks allows you to change scenarios easily. It does this by allowing
you to compose “scenarios” out of different mock files.

Here is the mock manifest for a default scenario and a `someItems` scenario:

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

Here is the default `empty.json` file:

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

Here is the `someItems.json` file:

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

As you can see when the user is in the default scenario a `GET` request to
`/customer/123/cart` will return the response defined in `empty.json`. When
the user is in the `someItems` scenario the response defined in
`someItems.json` will be returned.

As you start to scale your application, the power of composable scenarios
will become apparent. The folder structure and the mock manifest make it easy to
simulate a different states of your application.

Here you can see how we might have a number of scenarios which have the same
password authentication API call. By default the passcode authentication API
call will succeed. When the user is placed into the `passwordAuthFail` or
`passwordAuthLocked` scenarios, it will simulate a different state of the
application.

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

### How does it work

Multimocks uses `$httpBackend` to mock API calls. The developer creates mock
files and then composes them into different scenarios.

## That is awesome, but what does it cost?

Mocks need to be maintained, if an API changes then the mocks need to be
updated to match. If not, your tests may pass and when deployed the app
might not work.

You will still need to do of integration testing. I also recommend the
“Demo approach”.
This means that once code is ready for deployment the developer demos the
feature to stakeholders (include someone deeply familiar with the feature
requirements). This usually flags up any issues.

## Help make Multimocks better

We :heart: pull requests!
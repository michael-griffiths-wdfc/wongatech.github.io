---
layout: post
title: Building, Testing and Maintaining a Website Platform
author: edconolly
---
Over the past year we have had to change the way we look at software development
in the front-end team. We have expanded the same platform to deliver multiple
products in several countries, this has made us rethink the way we deliver our
software.

Within the team we maintain a “keep it simple” approach to pretty much
everything we do. Simple within a coding context really means easy to
understand. Good naming conventions, spacing and comments is a good place to
start. Importantly we try and catch over engineered solutions as early as
possible. We adhere to the agile philosophy of developing for the requirements
that have been presented, minimising the amount of code written and reducing the
amount of overall bugs in the system.

[TDD](http://en.wikipedia.org/wiki/Test-driven_development)
in conjunction with an adaption of the [SOLID design
principles](http://en.wikipedia.org/wiki/SOLID_(object-oriented_design)) to
apply to Drupal gives us the basis for better structured and more maintainable
code. An emphasis is placed on single responsibility, small, testable units of
code. These reusable units help us to reduce code wastage and help the overall
system take a more simplistic shape.

As our code base and team size grows keeping to these principles become more
important. Our keep it simple mantra is upheld most importantly within our tests
as we keep them explicit and easy to debug. It is the one place where we don’t
look for code reuse. Our tests do not make use of helper methods apart from the
work done by the setUp() method. Using the [Arrange, Act, Assert unit testing
pattern](http://c2.com/cgi/wiki?ArrangeActAssert) helps to keep test structure
and make code smells more obvious.

Gone are the days of "over the wall" QAing where a developer produces code and
somebody else verifies the quality. We have a 3 tier testing strategy which
involves Unit tests, Service tests and Platform tests.

Our Unit tests are written using PHPUnit, we don’t do much mocking on the Drupal
platform (apart from the API responses from the backend). Tthis is mainly due to
limitations of a framework that does not make use of dependency injection (DI).
The advantages are still the same however - quick feedback times, code coverage
metrics, code confidence .etc. Our new PHP platform makes use of DI and all of
the mocking capabilities of PHPUnit. We rely on mocks and stubs to isolate the
System Under Test (SUT). Our software engineers are required to write unit tests
as part of the development process and no code passes review without them.

Services tests, test the service in isolation. We exercise the website using
Selenium within a Selenium Grid framework. The backend is fully mocked so we are
only testing the frontend service. We are able to test the site on multiple
browsers, multiple operating systems and now emulating different mobile devices.
This gives us the confidence that at a high level our collaborative code is
functional at least within the confines of our test cases. We keep test times
down by using some of the features of AWS to achieve test run parallelisation.
Like the unit tests they are written and maintained by the feature developer.
This means that by the time code reaches a review stage a code reviewer will
have the original code, unit tests, and often a service test. The job of the
reviewer is then to make sure it adheres to our conventions and standards.

The platform tests are run against the full stack, both frontend and backend.
They are run using a page object module selenium framework. These are the
slowest running tests so we try to keep their numbers down and filter any
testing requirements into either the service tests or preferably into unit
tests. These end-to-end tests are our last line of defence and test all of the
integration points of the system.

It all comes back to testing, writing tests often means you are automatically
adhering to a lot of good software design principles and therefore inherently
producing better maintainable code. Our service level CI platform gives us
feedback within a few minutes about the overall health of the front-end platform
and reduces the overall cost and time to the business. These tests also act as
contract to the product owners as a stamp of authority on the functional
capability of the platform. Time that would be spent debugging issues further
along the CI pipeline is time that can be spent innovating with new or existing
technology to improve the development and testing process, all with the ultimate
aim to provide a better service to both our internal and external customers.

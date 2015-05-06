---
layout: post
title: LightBDD at Wonga
author: wojciechkotlarski
---
![logo](/images/lightbdd-at-wonga-logo.png)

After using [FitNesse](http://fitnesse.org/) and [SpecFlow](http://www.specflow.org), I created [LightBDD](https://github.com/Suremaker/LightBDD) as a developer-friendly way to write acceptance tests. LightBDD allows you to write acceptance tests entirely in code, using all the features of your IDE to create, maintain and refactor them, and generates reports that the whole team can use to ensure you're building the right thing.

## A bit of history: FitNesse

A few years ago, I was working on a project to create a web application for airport employees. We started experimenting with a new approach to writing and validating requirements.

The idea was that business analysts (BAs) would write requirements and we would automatically validate the system against them. The BAs were not programmers so we looked for tools which made it easy to write tests. We chose FitNesse, which provides a friendly test editor and  allows tests to be run from the UI. BAs could write new tests and run them immediately.

As our experience grew, we realised this approach had some pitfalls. What can be expressed in a test is limited to the set of conventions, keywords and methods supported by the underlying mapping code. Furthermore, the editor is just a plain text editor. It doesn't suggest method names or constructs that can be used in tests. If you don't know the underlying code, the only way to see what test syntax is available was to copy from other tests.

I don't recall a single instance of a BA using the FitNesse editor to write a test. Maybe they were too busy or maybe the tools created a barrier to entry. Developers wrote all tests and the mappings themselves. Writing tests this way was not fun. Any changes to the code meant all the corresponding tests had to be changed as well, in a basic plain text editor. Large changes to the tests or code were quite painful.

## A bit more history: SpecFlow

A few years later, on a different project, company and country, we tried [behaviour-driven development](http://dannorth.net/introducing-bdd/) (BDD) using SpecFlow to map acceptance criteria to code.

In BDD, acceptance criteria are structured as scenarios, e.g.:

```gherkin
Story: Returns go to stock

In order to keep track of stock
As a store owner
I want to add items back to stock when they're returned

Scenario 1: Refunded items should be returned to stock
Given a customer previously bought a black sweater from me
And I currently have three black sweaters left in stock
When he returns the sweater for a refund
Then I should have four black sweaters in stock
```

(Example from [Wikipedia](http://en.wikipedia.org/wiki/Behavior-driven_development).)

As our codebase and acceptance test suite grew, we realised that it was difficult to keep the acceptance criteria in plain text and the mapping code in sync. Also it made maintaining the test code hard, we couldn't see which test methods were in use and which could be removed for example.

Out of frustration with SpecFlow, I started working on a test framework that would allow us to:

- Keep all of our test definitions as clear as possible, so they would be readable and editable by people who aren't programmers
- Use all of the standard refactoring methods and IDE help to maintain those tests

## LightBDD

LightBDD is an open-source, lightweight framework for behaviour-driven development.

It's designed to strike a balance between writing test criteria so they can be understood by anyone and test code which is easy to maintain and extend.

In any BDD framework, acceptance tests have to be mapped to code at some point. In LightBDD we write the acceptance criteria directly in code so developers can use their IDE to maintain tests in an easy and natural way. There's no translation step, so there's only one version of the code and tests to maintain.

LightBDD is easy to learn and adopt and speeds up the overall development cycle.

### Easy to read scenarios

Here's how the example scenario above would look in LightBDD:

```csharp
[FeatureDescription(
@"In order to keep track of stock
As a store owner
I want to add items back to stock when they're returned")]
[TestFixture]
public partial class Returns_go_to_stock
{
    [Test]
    public void Refunded_items_should_be_returned_to_stock()
    {
        Runner.RunScenario(
            Given_a_customer_previously_bought_a_black_sweater_from_me,
            And_I_currently_have_three_black_sweaters_left_in_stock,
            When_he_returns_the_sweater_for_a_refund,
            Then_I_should_have_four_black_sweaters_in_stock);
    }
}
```

### Integration with existing tools

LightBDD is really just some conventions for structuring NUnit tests. You can use all the tools that Visual Studio and Resharper provide for refactoring, code analysis, e.g. finding unused methods, and executing tests.

Here's the implementation of the above scenario:

```csharp
public partial class Returns_go_to_stock : FeatureFixture
{
    private Item _sweater;
    private Shop _shop;

    [SetUp]
    public void SetUp()
    {
        _shop = new Shop();
    }

    private void Given_a_customer_previously_bought_a_black_sweater_from_me()
    {
        _sweater = new Item(ItemType.Sweater, Color.Black);
    }

    private void And_I_currently_have_three_black_sweaters_left_in_stock()
    {
        _shop.Stock.Add(new Item(ItemType.Sweater, Color.Black));
        _shop.Stock.Add(new Item(ItemType.Sweater, Color.Black));
        _shop.Stock.Add(new Item(ItemType.Sweater, Color.Black));
    }

    private void When_he_returns_the_sweater_for_a_refund()
    {
        _shop.Refund(_sweater);
    }

    private void Then_I_should_have_four_black_sweaters_in_stock()
    {
        Assert.That(
            _shop.Stock.Count(i => i.Type == ItemType.Sweater && i.Color == Color.Black),
            Is.EqualTo(4));
    }
}
```

### Reports in HTML, plain text and XML

![LightBDD Reports](/images/lightbdd-at-wonga-reports.png)


LightBDD generates [test reports in HTML](http://htmlpreview.github.io/?https://github.com/Suremaker/LightBDD/blob/master/ExampleSummaryFiles/FeaturesSummary.html). Anyone can read them and they contain a lot of useful detail. It can also generate plain text or XML for further processing.

## LightBDD at Wonga

At Wonga, LightBDD is a key tool for documenting the quality of our software. This is a requirement for our FCA authorisation process and LightBDD's reports serve as evidence that our code is correct    .

Product owners define scenarios in JIRA. Developers use those scenarios to write LightBDD tests. We use the [`Label`](https://github.com/Suremaker/LightBDD/wiki/02-Tests-Structure-and-Conventions) and [`Category`](https://github.com/Suremaker/LightBDD/wiki/02-Tests-Structure-and-Conventions) attributes to associate tests with JIRA stories and epics.

We run our acceptance tests in CI and archive the LightBDD reports. The HTML report is presented to POs as proof that a requested feature has been implemented properly. It's also used to map scenarios back to requirements and functions as evidence that we've built the right thing for our customers. XML reports are used to collect statistics on quality across projects.

## Read more

If LightBDD sounds interesting, visit the [project page](https://github.com/Suremaker/LightBDD) on GitHub.

It's also worth checking out the [wiki](https://github.com/Suremaker/LightBDD/wiki), which contains a lot more detail about LightBDD's features.

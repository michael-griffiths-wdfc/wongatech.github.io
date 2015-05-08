---
title: Dependency Injection on Drupal 6
author: maurogadaleta
---

[Dependency injection](http://martinfowler.com/articles/injection.html) is a
pattern that promotes loose coupling between classes and subsystems, creating
components that are easy to unit test in isolation. Instead of hard-coding the
names of other classes and objects, each unit accepts dependencies as
parameters.

At Wonga, we maintain a legacy Drupal 6 application. Drupal is an excellent CMS,
but to extend it you must write custom modules which are tightly-coupled to the
framework and difficult to test without a web server and a database.

This post will help you to refactor your Drupal 6 code base step-by-step using
objected-oriented paradigm and will help you to configure the Symfony dependency
injection container in this framework.

## Step 1: Installing dependencies through Composer

Composer is now the standard way to install dependencies for your
PHP applications.

We are going to use these two components:

- `Symfony/DependencyInjection`: Symfony's dependency injection container
- `Symfony/Config`: Provides several tools to help, find, load, combine,
  autofill and validate configuration values of any kind (YAML, XML, INI, etc)

We'll use the following `composer.json` configuration file:

```json
{
  "require": {
    "symfony/dependency-injection": "2.6.*@dev",
    "symfony/config": "2.6.*@dev"
  }
}
```

## Step 2: Understanding our new OO structure/architecture

Using the right directory structure makes it easy for developers to understand
and contribute to our application. Symfony's implementation of the service
locator strategy creates a level of indirection between a class and its
dependencies and provides a smoother migration path away from legacy code.

We'll use the following directory structure:

```
sites/all/modules/custom/GreeterBundle/
├── DependencyInjection
│   └── ContainerManager.php
├── Resources
│   └── config
│       └── services.yml
└── Services
    └── Greeter.php
```

Our code sits within a Drupal custom module, but is structured like a Symfony
bundle.

## Step 3: A simple service to inject

Our `Greeter` service takes a string (`name = "world"`) in the constructor and
contains a very simple public method, `greet`, which returns a familiar
greeting.

```php
<?php
namespace GreeterBundle\Services\Greeter;

class Greeter
{
  /**
   * Name to greet
   */
  private $name;

  /**
   * Constructor
   */
  public function __construct($name)
  {
    $this->name = $name;
  }

  /**
   * @return Greeting text
   */
  public function greet()
  {
    return "Hello, {$this->name}";
  }
}
```

## Step 4: Setting up the dependency injection container

The `ContainerManager` class provides us with a simple way to get a handle to
the DI container.

```php
<?php
namespace GreeterBundle\DependencyInjection;

// Including composer autoloader
require_once '/path/to/composer/autoload.php';

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;
use Symfony\Component\Config\FileLocator;

class ContainerManager
{
  /**
   * @var ContainerBuilder
   */
  private $container;

  /**
   * Constructor
   */
  public function __construct()
  {
    $this->container = new ContainerBuilder();
    $this->register();
  }

  private function register()
  {
    $fileLocator = new FileLocator(__DIR__ . '/../Resources/config');
    $loader = new YamlFileLoader($this->container, $fileLocator);
    $loader->load('services.yml');
  }

  /**
   * @return ContainerBuilder
   */
  public function getContainer()
  {
    return $this->container;
  }
}
```

The `services.yml` file looks like this:

```yaml
parameters:
  greeter.class: GreeterBundle\Services\Greeter
  greeter.name: "world"
services:
  greeter:
    class: "%greeter.class%"
    arguments: ["%greeter.name%"]
```

We could inject any value as a parameters to the service being constructed,
including other services, configuration parameters, 3rd party libraries, etc.
Take a look at the [Symfony Dependency Injection Component
documentation](http://symfony.com/doc/current/components/dependency_injection/introduction.html)
to check more advanced configuration.

## Step 5: Inject my service into a Drupal module function

We'll set up a Drupal page which uses the dependency injection container to get
our new service. To do this we create some routing configuration and a callback
function. The route configuration will use `ContainerManager` to get first the
container then our service.

```php
<?php
use GreeterBundle\DependencyInjection\ContainerManager;
use GreeterBundle\Services\Greeter;

// ...

$containerManager = new ContainerManager();
$container = $containerManager->getContainer();

$route[‘/homepage’] = array(
  'page callback' => 'greeter_module_function_homepage',
  'page arguments' => array($container->get('greeter')),
  'type' => MENU_CALLBACK,
);

// ...

function greeter_module_function_homepage(Greeter $greeter) {
  $result[`title`] = `Homepage!`;
  $result['greeting'] = $greeter->greet();
  return $result;
}
```

## Step 6: Unit testing in isolation

Unit testing in isolation is one of the most important DI pattern features. I’m
going to show you how we can unit test this Drupal modules mocking the
`Greeter` class.

```php
<?php
use GreeterBundle\Services\Greeter;

class ModuleFunctionHomePageTest extends PHPUnit_Framework_TestCase {
  public function testGreeterModuleFunctionHomePage() {
     // Arrange
     $greeting = "Hello world";
     $mockGreeter
        ->expects($this->once())
        ->method('name')
        ->will($this->returnValue($greeting));

     // Act
     $result = greeter_module_function_homepage($mockGreeter);

     // Assert
     $this->assertEquals($result['title'], 'Homepage!');
     $this->assertEquals($result['greeting'], $greeting);
  }
}
```

The `testGreeterModuleFunctionHomePage` test method calls the Drupal module
callback function with the mocked service and asserts that the return value is
the greeting returned by our service. This test requires no web server or
database and will be quicker to run and more stable as a result.

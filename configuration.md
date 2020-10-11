<h1 id="doc-title">Configuration</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Application Builders](#application-builders)
   1. [Modules](#modules)
2. [Components](#components)
   1. [Binders](#component-binders)
   2. [Routes](#component-routes)
   3. [Middleware](#component-middleware)
   4. [Console Commands](#component-console-commands)
   5. [Validator](#component-validator)
   6. [Exception Handler](#component-exception-handler)
3. [Adding Custom Components](#adding-custom-components)
4. [Reading From Configs](#reading-from-configs)
   1. [Reading PHP Files](#reading-php-files)
   2. [Reading JSON Files](#reading-json-files)
   3. [Custom File Readers](#custom-file-readers)
5. [Global Configuration](#global-configuration)
   1. [Building The Global Configuration](#building-the-global-configuration)

</div>

</nav>

<h2 id="application-builders">Application Builders</h2>

Application builders provide an easy way to configure your application's components, eg adding binders, routes, global middleware, console commands, validators, and more.  [Components](#components) are pieces of your application that are shared across modules (chunks of your domain).  If you are running a site where users can buy books, you might have a user module, a book module, and a shopping cart module.  Each of these modules will have separate binders, routes, console commands, etc.  So, why not bundle all the configuration logic by module?

Let's look at an example of a module:

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Console\Commands\Command;
use Aphiria\Console\Commands\CommandRegistry;
use Aphiria\Framework\Application\AphiriaComponents;
use Aphiria\Routing\Builders\RouteCollectionBuilder;

final class UserModule implements IModule
{
    // Gives us a fluent way to configure Aphiria components
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        $this->withBinders($appBuilder, [new UserBinder()])
            ->withRoutes($appBuilder, function (RouteCollectionBuilder $routes) {
                $routes->get('users/:id')
                    ->mapsToMethod(UserController::class, 'getUserById');
            })
            ->withCommands($appBuilder, function (CommandRegistry $commands) {
                $commands->registerCommand(
                    new Command('report:generate'),
                    GenerateUserReportCommandHandler::class
                );
            });
    }
}
```

Here's the best part of how Aphiria is built - all components, even Aphiria-provided components for things like binders, routes, console commands, etc, are not first-class citizens.  They're just normal components, which means it's trivial to [configure another library in place of any Aphiria libraries](#adding-custom-components) if you so choose.

<h3 id="modules">Modules</h3>

To register a module, you can use the `AphiriaComponents` trait:

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Framework\Application\AphiriaComponents;

final class App
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        $this->withModules($appBuilder, new MyModule());

        // Or register many modules

        $this->withModules($appBuilder, [new MyModule1(), new MyModule2()]);
    }
}
```

<h2 id="components">Components</h2>

A component is a piece of your application that is shared across business domains.  Below, we'll go over the components that are bundled with Aphiria, and some decoration methods to help configure them.

<h3 id="component-binders">Binders</h3>

You can configure your module to require [binders](dependency-injection.md#binders).

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Framework\Application\AphiriaComponents;

final class UserModule implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        // Add a binder
        $this->withBinders($appBuilder, new UserBinder());

        // Or use an array of binders
        $this->withBinders($appBuilder, [new UserBinder()]);
    }
}
```

<h3 id="component-routes">Routes</h3>

You can register [routes](routing.md) for your module, and you can enable route attributes.

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Framework\Application\AphiriaComponents;
use Aphiria\Routing\Builders\RouteCollectionBuilder;

final class UserModule implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        // Add some routes
        $this->withRoutes($appBuilder, function (RouteCollectionBuilder $routes) {
            $routes->get('users/:id')
                ->mapsToMethod(UserController::class, 'getUserById');
        });

        // Enable route attributes
        $this->withRouteAttributes($appBuilder);
    }
}
```

<h3 id="component-middleware">Middleware</h3>

Some modules might need to add global [middleware](middleware.md) to your application.

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Framework\Application\AphiriaComponents;
use Aphiria\Middleware\MiddlewareBinding;

final class UserModule implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        // Add global middleware (executed before each route)
        $this->withGlobalMiddleware($appBuilder, new MiddlewareBinding(Cors::class));

        // Or use an array of bindings
        $this->withGlobalMiddleware($appBuilder, [new MiddlewareBinding(Cors::class)]);

        // Or with a priority (lower number = higher priority)
        $this->withGlobalMiddleware($appBuilder, new MiddlewareBinding(Cors::class), 1);
    }
}
```

<h3 id="component-console-commands">Console Commands</h3>

You can register [console commands](console.md#creating-commands), and enable command attributes from your modules.

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Console\Commands\Command;
use Aphiria\Console\Commands\CommandRegistry;
use Aphiria\Framework\Application\AphiriaComponents;

final class UserModule implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        // Add console commands
        $this->withCommands($appBuilder, function (CommandRegistry $commands) {
            $commands->registerCommand(
                new Command('report:generate'),
                GenerateUserReportCommandHandler::class
            );
        });

        // Enable command attributes
        $this->withCommandAttributes($appBuilder);

        // Register built-in framework commands
        $this->withFrameworkCommands($appBuilder);

        // Register built-in framework commands, but exclude certain ones
        $this->withFrameworkCommands($appBuilder, ['app:serve']);
    }
}
```

<h3 id="component-validator">Validator</h3>

You can also configure [constraints](validation.md#constraints) for your models and enable [validator attributes](validation.md#validation-attributes).

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Framework\Application\AphiriaComponents;
use Aphiria\Validation\Builders\ObjectConstraintsRegistryBuilder;
use Aphiria\Validation\Constraints\EmailConstraint;

final class UserModule implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        // Add constraints to a class
        $this->withObjectConstraints($appBuilder, function (ObjectConstraintsRegistryBuilder $objectConstraintsBuilder) {
            $objectConstraintsBuilder->class(User::class)
                ->hasPropertyConstraints('email', new EmailConstraint());
        });

        // Enable validator attributes
        $this->withValidatorAttributes($appBuilder);
    }
}
```

<h3 id="component-exception-handler">Exception Handler</h3>

Exceptions may be mapped to [custom problem details](exception-handling.md#custom-problem-details-mappings) and [PSR-3 log levels](exception-handling.md#exception-log-levels).

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Console\Output\IOutput;
use Aphiria\Console\StatusCodes;
use Aphiria\Framework\Application\AphiriaComponents;
use Aphiria\Net\Http\HttpStatusCodes;
use Aphiria\Net\Http\IRequest;
use Aphiria\Net\Http\IResponseFactory;
use Psr\Log\LogLevel;

final class UserModule implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        // Add a custom problem details status code for an exception
        $this->withProblemDetails(
            $appBuilder,
            UserNotFoundException::class,
            status: HttpStatusCodes::NOT_FOUND
        );

        // Add a completely custom problem details mapping for an exception
        $this->withProblemDetails(
            $appBuilder,
            OverdrawnException::class,
            'https://example.com/errors/overdrawn', // Type
            'This account is overdrawn', // Title
            fn ($ex) => "Account {$ex->accountId} is overdrawn by {$ex->overdrawnAmount}", // Detail
            HttpStatusCodes::BAD_REQUEST, // Status
            fn ($ex) => "https://example.com/accounts/{$ex->accountId}/errors/{$ex->id}", // Instance
            fn ($ex) => ['overdrawnAmount' => $ex->overdrawnAmount] // Extensions
        );

        // Add a custom console output writer for an exception
        $this->withConsoleExceptionOutputWriter(
            $appBuilder,
            UserNotFoundException::class,
            function (UserNotFoundException $ex, IOutput $output) {
                $output->writeln('Missing user');

                return StatusCodes::FATAL;
            }
        );

        // Add a custom PSR-3 log level for an exception
        $this->withLogLevelFactory(
            $appBuilder,
            UserCorruptedException::class,
            fn (UserCorruptedException $ex) => LogLevel::CRITICAL
        );
    }
}
```

<h2 id="adding-custom-components">Adding Custom Components</h2>

You can add your own custom components to application builders.  They typically have `with*()` methods to let you configure the component, and a `build()` method (called internally) that actually finishes building the component.

> **Note:** Binders aren't dispatched until just before `build()` is called on the components.  This means you can't inject dependencies from binders into your components - they won't have been bound yet.  So, if you need any dependencies inside the `build()` method, use the DI container to resolve them.

Let's say you prefer to use Symfony's router, and want to be able to add routes from your modules.  This requires a few simple steps:

1. Create a binder for the Symfony services
3. Create a component to let you add routes from modules
2. Register the binder and component to your app
4. Start using the component

First, let's create a binder for the router so that the DI container can resolve it:

```php
use Aphiria\DependencyInjection\Binders\Binder;
use Aphiria\DependencyInjection\IContainer;
use Symfony\Component\Routing\Matcher\UrlMatcher;
use Symfony\Component\Routing\Matcher\UrlMatcherInterface;
use Symfony\Component\Routing\RequestContext;
use Symfony\Component\Routing\RouteCollection;

final class SymfonyRouterBinder extends Binder
{
    public function bind(IContainer $container): void
    {
        $routes = new RouteCollection();
        $requestContext = new RequestContext(/* ... */);
        $matcher = new UrlMatcher($routes, $requestContext);

        $container->bindInstance(RouteCollection::class, $routes);
        $container->bindInstance(UrlMatcherInterface ::class, $matcher);
    }
}
```

Next, let's define a component to let us add routes.

```php
use Aphiria\Api\Application;
use Aphiria\Application\IComponent;
use Aphiria\DependencyInjection\IContainer;
use Aphiria\Net\Http\IRequestHandler;
use Symfony\Component\Routing\Route;
use Symfony\Component\Routing\RouteCollection;

final class SymfonyRouterComponent implements IComponent
{
    private array $routes = [];

    public function __construct(private IContainer $container) {}

    public function build(): void
    {
        $routes = $this->container->resolve(RouteCollection::class);

        foreach ($this->routes as $name => $route) {
            $routes->add($name, $route);
        }

        // Assume we've created a request handler that uses the Symfony route matcher
        $this->container->for(Application::class, function (IContainer $container) {
            $container->bindInstance(IRequestHandler::class, new SymfonyRouterRequestHandler());
        });
    }

    // Define a method for adding routes from modules
    public function withRoutes(string $name, Route $route): self
    {
        $this->routes[$name] = $route;

        return $this;
    }
}
```

Let's register the binder and component to our app:

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\DependencyInjection\IContainer;
use Aphiria\Framework\Application\AphiriaComponents;

final class App implements IModule
{
    use AphiriaComponents;

    public function __construct(private IContainer $container) {}

    public function build(IApplicationBuilder $appBuilder): void
    {
        $this->withComponent($appBuilder, new SymfonyRouterComponent($this->container))
            ->withBinders($appBuilder, new SymfonyRouterBinder());
    }
}
```

All that's left is to start using the component from a module:

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Symfony\Component\Routing\Route;

final class MyModule implements IModule
{
    public function build(IApplicationBuilder $appBuilder): void
    {
        $appBuilder->getComponent(SymfonyRouterComponent::class)
            ->withRoutes('GetUserById', new Route('users/{id}'));
    }
}
```

<h2 id="reading-from-configs">Reading From Configs</h2>

Configs allow you to store changeable values that power your application.  Unlike environment variables, they do not typically change between environments.  Configurations must implement `IConfiguration`, which provides the following methods:

```php
// Get the value as an array
$config->getArray('foo');

// Get the value as a boolean
$config->getBool('foo');

// Get the value as a float
$config->getFloat('foo');

// Get the value as an integer
$config->getInt('foo');

// Get the value as a string
$config->getString('foo');

// Get the raw value
$config->getValue('foo');

// Try to get the value as an array
$config->tryGetArray('foo', $value);

// Try to get the value as a boolean
$config->tryGetBool('foo', $value);

// Try to get the value as a float
$config->tryGetFloat('foo', $value);

// Try to get the value as an integer
$config->tryGetInt('foo', $value);

// Try to get the value as a string
$config->tryGetString('foo', $value);

// Try to get the raw value
$config->tryGetValue('foo', $value);
```

<h3 id="reading-php-files">Reading PHP Files</h3>

Let's say you have a PHP config array that looks like this:

```php
return [
    'api' => [
        'supportedLanguages' => ['en', 'es']
    ]
];
```

You can load this PHP file into a configuration object:

```php
use Aphiria\Application\Configuration\PhpConfigurationFileReader;

$config = (new PhpConfigurationFileReader)->readConfiguration('config.php');
```

Grab the supported languages by using `.` as a delimiter for nested sections:

```php
$supportedLanguages = $config->getArray('api.supportedLanguages');
```

> **Note:**  Avoid using periods as keys in your configs.  If you must, you can change the delimiter character (eg to `:`) by passing it in as a second parameter to `readConfiguration()`.

<h3 id="reading-json-files">Reading JSON Files</h3>

Similarly, you can read JSON config files.

```php
use Aphiria\Application\Configuration\JsonConfigurationFileReader;

$config = (new JsonConfigurationFileReader)->readConfiguration('config.json');
```

<h3 id="custom-file-readers">Custom File Readers</h3>

You can create your own custom file reader.  Let's look at an example that reads from YAML files:

```php
use Aphiria\Application\Configuration\HashTableConfiguration;
use Aphiria\Application\Configuration\IConfiguration;
use Aphiria\Application\Configuration\IConfigurationFileReader;
use Aphiria\Application\Configuration\InvalidConfigurationFileException;
use Symfony\Component\Yaml\Yaml;

final class YamlConfigurationFileReader implements IConfigurationFileReader
{
    public function readConfiguration(string $path, string $pathDelimiter = '.'): IConfiguration
    {
        if (!file_exists($path)) {
            throw new InvalidConfigurationFileException("$path does not exist");
        }

        $hashTable = Yaml::parseFile($path);

        if (!\is_array($hashTable)) {
            throw new InvalidConfigurationFileException("Failed to convert YAML in $path to an array");
        }

        return new HashTableConfiguration($hashTable, $pathDelimiter);
    }
}
```
  
<h2 id="global-configuration">Global Configuration</h2>

`GlobalConfiguration` is a static class that can access values from multiple configurations registered via `GlobalConfiguration::addConfigurationSource()`.  It is the most convenient way to read configuration values from places like [binders](dependency-injection.md#binders).  Let's look at its methods:

```php
use Aphiria\Application\Configuration\GlobalConfiguration;

// Get the value as an array
GlobalConfiguration::getArray('foo');

// Get the value as a boolean
GlobalConfiguration::getBool('foo');

// Get the value as a float
GlobalConfiguration::getFloat('foo');

// Get the value as an integer
GlobalConfiguration::getInt('foo');

// Get the value as a string
GlobalConfiguration::getString('foo');

// Get the raw value
GlobalConfiguration::getValue('foo');

// Try to get the value as an array
GlobalConfiguration::tryGetArray('foo', $value);

// Try to get the value as a boolean
GlobalConfiguration::tryGetBool('foo', $value);

// Try to get the value as a float
GlobalConfiguration::tryGetFloat('foo', $value);

// Try to get the value as an integer
GlobalConfiguration::tryGetInt('foo', $value);

// Try to get the value as a string
GlobalConfiguration::tryGetString('foo', $value);

// Try to get the raw value
GlobalConfiguration::tryGetValue('foo', $value);
```

These methods mimic the `IConfiguration` interface, but are static.  Like `IConfiguration`, you can use `.` as a delimiter between sections.  If you have 2 configuration sources registered, `GlobalConfiguration` will attempt to find the path in the first registered source, and, if it's not found, the second source.  If no value is found, the `get*()` methods will throw a `MissingConfigurationValueException`, and the `tryGet*()` methods will return `false`.

<h3 id="building-the-global-configuration">Building The Global Configuration</h3>

`GlobalConfigurationBuilder` simplifies configuring different sources and building your global configuration.  In this example, we're loading from a PHP file, a JSON file, and environment variables:

```php
use Aphiria\Application\Configuration\GlobalConfigurationBuilder;

$globalConfigurationBuilder = new GlobalConfigurationBuilder();
$globalConfigurationBuilder->withPhpFileConfigurationSource('config.php')
    ->withJsonFileConfigurationSource('config.json')
    ->withEnvironmentVariables()
    ->build();
```

> **Note:** The reading of files and environment variables is deferred until `build()` is called.

After `build()` is called, you can start accessing the values from `config.php`, `config.json`, and environment variables via `GlobalConfiguration`.

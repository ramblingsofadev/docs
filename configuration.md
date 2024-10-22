<h1 id="doc-title">Configuration</h1>

<nav class="toc-nav">

<div class="toc-nav-contents">

<h2 id="table-of-contents">Table of Contents</h2>

<ol>
<li><a href="#application-builders">Application Builders</a><ol>
<li><a href="#modules">Modules</a></li>
</ol>
</li>
<li><a href="#components">Components</a><ol>
<li><a href="#component-binders">Binders</a></li>
<li><a href="#component-routes">Routes</a></li>
<li><a href="#component-middleware">Middleware</a></li>
<li><a href="#component-console-commands">Console Commands</a></li>
<li><a href="#component-authentiators">Authenticators</a></li>
<li><a href="#component-authorities">Authorities</a></li>
<li><a href="#component-validator">Validator</a></li>
<li><a href="#component-exception-handler">Exception Handler</a></li>
</ol>
</li>
<li><a href="#adding-custom-components">Adding Custom Components</a></li>
<li><a href="#reading-from-configs">Reading From Configs</a><ol>
<li><a href="#reading-php-files">Reading PHP Files</a></li>
<li><a href="#reading-json-files">Reading JSON Files</a></li>
<li><a href="#reading-yaml-files">Reading YAML Files</a></li>
<li><a href="#custom-file-readers">Custom File Readers</a></li>
</ol>
</li>
<li><a href="#global-configuration">Global Configuration</a><ol>
<li><a href="#building-the-global-configuration">Building The Global Configuration</a></li>
</ol>
</li>
<li><a href="#custom-applications">Custom Applications</a></li>
</ol>

</div>

</nav>

<h2 id="application-builders">Application Builders</h2>

Application builders provide an easy way to configure your application.  They are passed into [modules](#modules) where you can decorate them with the [components](#components) a part of your business domain needs (eg routes, binders, global middleware, console commands, validators, etc).  For example, if you are running a site where users can buy books, you might have a user module, a book module, and a shopping cart module.  Each of these modules will have separate binders, routes, console commands, etc.  So, why not bundle all the configuration logic by module?

Let's look at an example of a module:

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Console\Commands\Command;
use Aphiria\Console\Commands\CommandRegistry;
use Aphiria\Framework\Application\AphiriaModule;
use Aphiria\Routing\RouteCollectionBuilder;

final class UserModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        $this->withBinders($appBuilder, new UserServiceBinder())
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

Here's the best part of how Aphiria was built - there's nothing special about Aphiria-provided components.  You can [write your own components](#adding-custom-components) to be just as powerful and easy to use as Aphiria's.

Another great thing about Aphiria's application builders is that they allow you to abstract away the runtime of your application (eg PHP-FPM or Swoole) without having to touch your domain logic.  We'll get into more details on how to do this [below](#custom-applications).

<h3 id="modules">Modules</h3>

Either extend `AphiriaModule` or use the `AphiriaComponents` trait to register a module:

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Framework\Application\AphiriaModule;

final class GlobalModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
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
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Framework\Application\AphiriaModule;

final class UserModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        // Add a binder
        $this->withBinders($appBuilder, new UserServiceBinder());

        // Or use an array of binders
        $this->withBinders($appBuilder, [new UserServiceBinder()]);
    }
}
```

<h3 id="component-routes">Routes</h3>

You can manually register [routes](routing.md) for your module, and you can enable route attributes.

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Framework\Application\AphiriaModule;
use Aphiria\Routing\RouteCollectionBuilder;

final class UserModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        // Manually add some routes
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
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Framework\Application\AphiriaModule;
use Aphiria\Middleware\MiddlewareBinding;

final class UserModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        // Add global middleware (executed before each route)
        $this->withGlobalMiddleware($appBuilder, new MiddlewareBinding(Cors::class));

        // Or use an array of bindings
        $this->withGlobalMiddleware($appBuilder, [new MiddlewareBinding(Cors::class)]);

        // Or with a priority (lower number == higher priority)
        $this->withGlobalMiddleware($appBuilder, new MiddlewareBinding(Cors::class), 1);
    }
}
```

<h3 id="component-console-commands">Console Commands</h3>

You can manually register [console commands](console.md#creating-commands), and enable command attributes from your modules.

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Console\Commands\Command;
use Aphiria\Console\Commands\CommandRegistry;
use Aphiria\Framework\Application\AphiriaModule;

final class UserModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        // Manually add console commands
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

<h3 id="component-authenticators">Authenticators</h3>

Aphiria provides methods for configuring your [authenticator](authentication.md).

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Authentication\AuthenticationScheme;
use Aphiria\Authentication\Schemes\BasicAuthenticationOptions;
use Aphiria\Authentication\Schemes\CookieAuthenticationOptions;
use Aphiria\Framework\Application\AphiriaModule;

final class GlobalModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        // Register an authentication scheme
        $this->withAuthenticationScheme($appBuilder, new AuthenticationScheme(
            'basic',
            BasicAuthenticationHandler::class,
            new BasicAuthenticationOptions(realm: 'example.com', claimsIssuer: 'https://example.com')
        ));
        
        // Register a default authentication scheme
        $this->withAuthenticationScheme($appBuilder, new AuthenticationScheme(
            'cookie',
            CookieAuthenticationHandler::class,
            new CookieAuthenticationOptions(cookieName: 'authToken', claimsIssuer: 'https://example.com')
        ), true);
    }
}
```

<h3 id="component-authorities">Authorities</h3>

Customizing your [authority](authorization.md) is also simple.

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Authorization\AuthorizationPolicy;
use Aphiria\Authorization\Requirements\RolesRequirement;
use Aphiria\Authorization\Requirements\RolesRequirementHandler;
use Aphiria\Framework\Application\AphiriaModule;

final class GlobalModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        // Register an authorization policy
        $this->withAuthorizationPolicy($appBuilder, new AuthorizationPolicy(
            'requires-admin',
            new RolesRequirement('admin'),
            'cookie'
        ));
        
        // Register an authorization requirement handler
        $this->withAuthorizationRequirementHandler(
            $appBuilder,
            RolesRequirement::class,
            new RolesRequirementHandler()
        );
    }
}
```

<h3 id="component-validator">Validator</h3>

You can also manually configure [constraints](validation.md#constraints) for your models and enable [validator attributes](validation.md#creating-a-validator).

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Framework\Application\AphiriaModule;
use Aphiria\Validation\Constraints\EmailConstraint;
use Aphiria\Validation\ObjectConstraintsRegistryBuilder;

final class UserModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        // Manually add constraints to a class
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
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Console\Output\IOutput;
use Aphiria\Console\StatusCodes;
use Aphiria\Framework\Application\AphiriaModule;
use Aphiria\Net\Http\HttpStatusCode;
use Psr\Log\LogLevel;

final class UserModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        // Add a custom problem details status code for an exception
        $this->withProblemDetails(
            $appBuilder,
            UserNotFoundException::class,
            status: HttpStatusCode::NotFound
        );

        // Add a completely custom problem details mapping for an exception
        $this->withProblemDetails(
            $appBuilder,
            OverdrawnException::class,
            type: 'https://example.com/errors/overdrawn',
            title: 'This account is overdrawn',
            detail: fn ($ex) => "Account {$ex->accountId} is overdrawn by {$ex->overdrawnAmount}",
            status: HttpStatusCode::BadRequest,
            instance: fn ($ex) => "https://example.com/accounts/{$ex->accountId}/errors/{$ex->id}",
            extensions: fn ($ex) => ['overdrawnAmount' => $ex->overdrawnAmount]
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

> **Note:** Binders aren't dispatched until just before `build()` is called on the components.  This means you can't inject dependencies from binders into your components - they won't have been bound yet.  So, if you need any dependencies inside the `build()` method, use the [DI container](dependency-injection.md) to resolve them.

Let's say you prefer to use Symfony's router, and want to be able to add routes from your modules.  This requires a few simple steps:

1. Create a binder for the Symfony services
2. Create a component to let you add routes from modules
3. Register the binder and component to your app
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

        // Tell our app to use a request handler that supports Symfony
        // Note: You'd have to write this request handler
        $this->container->for(Application::class, function (IContainer $container) {
            $container->bindInstance(IRequestHandler::class, new SymfonyRouterRequestHandler());
        });
    }

    // Define a method for adding routes from modules
    public function withRoute(string $name, Route $route): self
    {
        $this->routes[$name] = $route;

        return $this;
    }
}
```

Let's register the binder and component to our app:

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\DependencyInjection\IContainer;
use Aphiria\Framework\Application\AphiriaModule;

final class GlobalModule extends AphiriaModule
{
    public function __construct(private IContainer $container) {}

    public function configure(IApplicationBuilder $appBuilder): void
    {
        $this->withComponent($appBuilder, new SymfonyRouterComponent($this->container))
            ->withBinders($appBuilder, new SymfonyRouterBinder());
    }
}
```

All that's left is to start using the component from a module:

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Application\IModule;
use Symfony\Component\Routing\Route;

final class MyModule implements IModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        $appBuilder->getComponent(SymfonyRouterComponent::class)
            ->withRoute('GetUserById', new Route('users/{id}'));
    }
}
```

If you'd like a more fluent syntax like the Aphiria components, just use a trait:

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\DependencyInjection\Container;
use Symfony\Component\Routing\Route;

trait SymfonyComponents
{
    protected function withSymfonyRoute(IApplicationBuilder $appBuilder, string $name, Route $route): static
    {
        // Make sure the component is registered
        if (!$appBuilder->hasComponent(SymfonyRouterComponent::class)) {
            $appBuilder->withComponent(new SymfonyRouterComponent(Container::$globalInstance));
        }
        
        $appBuilder->getComponent(SymfonyRouterComponent::class)
            ->withRoute($name, $route);
            
        return $this;
    }
}
```

Then, use that trait inside your module:

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Application\IModule;
use Symfony\Component\Routing\Route;

final class MyModule implements IModule
{
    use SymfonyComponents;

    public function configure(IApplicationBuilder $appBuilder): void
    {
        $this->withSymfonyRoute($appBuilder, 'GetUserById', new Route('users/{id}'));
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

// Get the value as an object
$config->getObject('foo', fn (mixed $options): MyObject => new MyObject($options));

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

// Try to get the value as an object
$config->tryGetObject('foo', fn (mixed $options): MyObject => new MyObject($options), $value);

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

$config = new PhpConfigurationFileReader()->readConfiguration('config.php');
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

$config = new JsonConfigurationFileReader()->readConfiguration('config.json');
```

<h3 id="reading-YAML-files">Reading YAML Files</h3>

Aphiria supports reading YAML config files, too, as long as they parse to an associative array in PHP.

```php
use Aphiria\Application\Configuration\YamlConfigurationFileReader;

$config = new YamlConfigurationFileReader()->readConfiguration('config.yaml');
```

<h3 id="custom-file-readers">Custom File Readers</h3>

You can create your own custom file reader by implementing `IConfigurationFileReader`, which just needs to know how to convert the file contents to a PHP associative array.
  
<h2 id="global-configuration">Global Configuration</h2>

`GlobalConfiguration` is a static class that can access values from multiple configurations that were registered via `GlobalConfiguration::addConfigurationSource()`.  It is the most convenient way to read configuration values from places like [binders](dependency-injection.md#binders).  Let's look at its methods:

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

// Get the value as an object
GlobalConfiguration::getObject('foo', fn (mixed $options): MyObject => new MyObject($options));

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

// Try to get the value as an object
GlobalConfiguration::tryGetObject('foo', fn (mixed $options): MyObject => new MyObject($options), $value);

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

<h2 id="custom-applications">Custom Applications</h2>

This is more of an advanced topic.  Applications are specific to their runtimes, eg PHP-FPM or Swoole.  They typically take the input (eg an HTTP request or console input) and pass it to  a "gateway" object (eg `ApiGateway` or `ConsoleGateway`), which is the highest layer of application code that is agnostic to the PHP runtime.  So, if you switch from PHP-FPM to Swoole, you'd have to change the `IApplication` instance you're running, but the gateway would not have to change because it does not care what the PHP runtime is.

By default, API and console applications are built with `SynchronousApiApplicationBuilder` and `ConsoleApplicationBuilder`, respectively.  Which application builder you're using depends on the `APP_BUILDER_API` and `APP_BUILDER_CONSOLE` environment variables set in your _.env_ file.  Application builder classes return a simple `IApplication` interface that looks like this:

```php
interface IApplication
{
    public function run(): int;
}
```

The simplest way to change the `IApplication` you're running is to create your own `IApplicationBuilder` and update the appropriate environment variable to use it.  For example, let's say we wanted to switch our Aphiria app to use Swoole instead:

```php
use Aphiria\Application\IApplication;
use Aphiria\Net\Http\IRequest;
use Aphiria\Net\Http\IRequestHandler;
use Aphiria\Net\Http\IResponse;
use Swoole\Http\Request;
use Swoole\Http\Response;
use Swoole\Http\Server;

final class SwooleApplication implements IApplication
{
    public function __construct(private Server $server, private IRequestHandler $apiGateway) {}

    public function run(): int
    {
        $server->on('request', function (Request $swooleRequest, Response $swooleResponse) use ($this) {
            $aphiriaRequest = $this->createAphiriaRequest($swooleRequest);
            $aphiriaResponse = $this->apiGateway->handle($aphiriaRequest);
            $this->copyToSwooleResponse($aphiriaResponse, $swooleResponse);
        });
        $server->start();
        
        return 0;
    }
    
    private function copyToSwooleResponse(IResponse $aphiriaResponse, Response $swooleResponse): void
    {
        // ...
    }
    
    private function createAphiriaRequest(Request $request): IRequest
    {
        // ...
    }
}
```

Next, create an `IApplicationBuilder` that builds an instance of our `SwooleApplication`:

```php
namespace App;

use Aphiria\Application\IApplication;
use Aphiria\Application\ApplicationBuilder;
use Aphiria\DependencyInjection\IServiceResolver;
use Aphiria\Net\Http\IRequestHandler;
use Swoole\Http\Server;

final class SwooleApplicationBuilder extends ApplicationBuilder
{
    public function __construct(private IServiceResolver $serviceResolver) {}

    public function build(): SwooleApplication
    {
        $this->configureModules();
        $this->buildComponents();
        $server = new Server(/* ... */);
        
        return new SwooleApplication($server, $this->serviceResolver->resolve(IRequestHandler::class));
    }
}
```

Finally, update `APP_BUILDER_API` in your _.env_ file, and your application will now support running asynchronously via Swoole.

```dotenv
APP_BUILDER_API=\App\SwooleApplicationBuilder
```

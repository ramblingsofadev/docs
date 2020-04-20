<h1 id="doc-title">Application Builders</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
   1. [Modules](#modules)
2. [Components](#components)
   1. [Binders](#component-binders)
   2. [Routes](#component-routes)
   3. [Middleware](#component-middleware)
   4. [Console Commands](#component-console-commands)
   5. [Validator](#component-validator)
   6. [Serializer](#component-serializer)
   7. [Exception Handler](#component-exception-handler)
3. [Adding Custom Components](#adding-custom-components)

</div>

</nav>

<h2 id="basics">Basics</h2>

Application builders provide an easy way to configure your application's components, eg adding binders, routes, global middleware, console commands, validators, and more.  [Components](#components) are pieces of your application that are shared across modules (chunks of your domain).  If you are running a site where users can buy books, you might have a user module, a book module, and a shopping cart module.  Each of these modules will have separate binders, routes, console commands, etc.  So, why not bundle all the configuration logic by module?

Let's look at an example of a module:

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Console\Commands\CommandRegistry;
use Aphiria\Framework\Application\AphiriaComponents;
use Aphiria\Routing\Builders\RouteCollectionBuilder;

class UserModule implements IModule
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
                    new GenerateUserReportCommand(),
                    fn () => new GenerateUserReportCommandHandler()
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

class App
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

A component is a piece of your application that is shared across domains.  Below, we'll go over the components that are bundled with Aphiria, and some decoration methods to help configure them.

<h3 id="component-binders">Binders</h3>

You can configure your module to require [binders](binders.md).

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Framework\Application\AphiriaComponents;

class UserModule implements IModule
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

You can also configure your app to use a particular [binder dispatcher](binders.md#lazy-dispatching):

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Configuration\GlobalConfiguration;
use Aphiria\DependencyInjection\Binders\LazyBinderDispatcher;
use Aphiria\DependencyInjection\Binders\Metadata\Caching\FileBinderMetadataCollectionCache;
use Aphiria\Framework\Application\AphiriaComponents;

class App implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        if (\getenv('APP_ENV') === 'production') {
            $cachePath = GlobalConfiguration::getString('aphiria.binders.metadataCachePath');
            $cache = new FileBinderMetadataCollectionCache($cachePath);
        } else {
            $cache = null;
        }

        $this->withBinderDispatcher($appBuilder, new LazyBinderDispatcher($cache));
    }
}
```

<h3 id="component-routes">Routes</h3>

You can register [routes](routing.md) for your module, and you can enable route annotations.

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Framework\Application\AphiriaComponents;
use Aphiria\Routing\Builders\RouteCollectionBuilder;

class UserModule implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        // Add some routes
        $this->withRoutes($appBuilder, function (RouteCollectionBuilder $routes) {
            $routes->get('users/:id')
                ->mapsToMethod(UserController::class, 'getUserById');
        });

        // Enable route annotations
        $this->withRouteAnnotations($appBuilder);
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

class UserModule implements IModule
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

You can register [console commands](console.md#creating-commands), and enable command annotations from your modules.

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Console\Commands\CommandRegistry;
use Aphiria\Framework\Application\AphiriaComponents;

class UserModule implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        // Add console commands
        $this->withCommands($appBuilder, function (CommandRegistry $commands) {
            $commands->registerCommand(
                new GenerateUserReportCommand(),
                fn () => new GenerateUserReportCommandHandler()
            );
        });

        // Enable command annotations
        $this->withCommandAnnotations($appBuilder);
    }
}
```

<h3 id="component-validator">Validator</h3>

You can also configure [constraints](validation.md#constraints) for your models and enable [validator annotations](validation.md#validation-annotations).

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Framework\Application\AphiriaComponents;
use Aphiria\Validation\Builders\ObjectConstraintsRegistryBuilder;
use Aphiria\Validation\Constraints\EmailConstraint;

class UserModule implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        // Add constraints to a class
        $this->withObjectConstraints($appBuilder, function (ObjectConstraintsRegistryBuilder $objectConstraintsBuilder) {
            $objectConstraintsBuilder->class(User::class)
                ->hasPropertyConstraints('email', new EmailConstraint());
        });

        // Enable validator annotations
        $this->withValidatorAnnotations($appBuilder);
    }
}
```

<h3 id="component-serializer">Serializer</h3>

Your modules can configure [custom encoders](serialization.md#encoders) for your models.

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Framework\Application\AphiriaComponents;
use Aphiria\Serialization\Encoding\EncodingContext;
use Aphiria\Serialization\Encoding\IEncoder;

class UserModule implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        // Add an encoder to simplify serializing an object
        $this->withEncoder($appBuilder, User::class, new class() implements IEncoder {
            public function decode($userHash, string $type, EncodingContext $context): User
            {
                return new User($userHash['id'], $userHash['email']);
            }

            public function encode($user, EncodingContext $context)
            {
                return ['id' => $user->id, 'email' => $user->email];
            }
        });
    }
}
```

<h3 id="component-exception-handler">Exception Handler</h3>

Exceptions may be mapped to [custom HTTP responses](exception-handling.md#exception-responses) and [PSR-3 log levels](exception-handling.md#exception-log-levels).

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

class UserModule implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        // Add a custom HTTP response for an exception
        $this->withHttpExceptionResponseFactory(
            $appBuilder,
            UserNotFoundException::class,
            function (UserNotFoundException $ex, IRequest $request, IResponseFactory $responseFactory) {
                return $responseFactory->createResponse($request, HttpStatusCodes::HTTP_NOT_FOUND);
            }
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

class SymfonyRouterBinder extends Binder
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
use Aphiria\Net\Http\Handlers\IRequestHandler;
use Symfony\Component\Routing\Route;
use Symfony\Component\Routing\RouteCollection;

class SymfonyRouterComponent implements IComponent
{
    private IContainer $container;
    private array $routes = [];

    public function __construct(IContainer $container)
    {
        $this->container = $container;
    }

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

    // Our own method for adding routes
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

class App implements IModule
{
    use AphiriaComponents;

    private IContainer $container;

    public function __construct(IContainer $container)
    {
        $this->container = $container;
    }

    public function build(IApplicationBuilder $appBuilder): void
    {
        $this->withComponent($appBuilder, new SymfonyRouterComponent($this->container))
            ->withBinders($appBuilder, new SymfonyRouterBinder());
    }
}
```

All that's left is to start using it from a module:

```php
use Aphiria\Application\IModule;
use Aphiria\Application\Builders\IApplicationBuilder;
use Symfony\Component\Routing\Route;

class MyModule implements IModule
{
    public function build(IApplicationBuilder $appBuilder) : void
    {
        $appBuilder->getComponent(SymfonyRouterComponent::class)
            ->withRoutes('GetUserById', new Route('users/{id}'));
    }
}
```

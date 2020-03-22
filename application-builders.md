<h1 id="doc-title">Application Builders</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
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
use Aphiria\Routing\Builders\RouteBuilderRegistry;

class UserModule implements IModule
{
    // Gives us a fluent way to configure Aphiria components
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        $this->withBinders($appBuilder, [new UserBinder()]);

        $this->withRoutes($appBuilder, function (RouteBuilderRegistry $routeBuilders) {
            $routeBuilders->get('users/:id')
                ->toMethod(UserController::class, 'getUserById');
        });

        $this->withCommands($appBuilder, function (CommandRegistry $commands) {
            $commands->registerCommand(
                new GenerateUserReportCommand(),
                fn () => new GenerateUserReportCommandHandler()
            );
        });
    }
}
```

Application builders are agnostic to the types of applications they build as well as the components they configure, but Aphiria does provide `ApiApplicationBuilder` and `ConsoleApplicationBuilder`, along with various [components](#components), to simplify building API and console applications.

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

<h3 id="component-routes">Routes</h3>

You can register [routes](routing.md) for your module, and you can enable route annotations.

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Framework\Application\AphiriaComponents;
use Aphiria\Routing\Builders\RouteBuilderRegistry;

class UserModule implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        // Add some routes
        $this->withRoutes($appBuilder, function (RouteBuilderRegistry $routeBuilders) {
            $routeBuilders->get('users/:id')
                ->toMethod(UserController::class, 'getUserById');
        });

        // Enable route annotations
        $this->withRouteAnnotations($appBuilder);
    }
}
```

<h3 id="component-middleware">Middleware</h3>

Some modules might need to add global middleware to your application.

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

You can register console commands, and enable command annotations from your modules.

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

You can also configure constraints for your models and enable validator annotations.

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

Your modules can configure custom encoders for your models.

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

Exceptions may be mapped to custom HTTP responses and PSR-3 log levels.

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Framework\Application\AphiriaComponents;
use Aphiria\Net\Http\HttpStatusCodes;
use Aphiria\Net\Http\IHttpRequestMessage;
use Aphiria\Net\Http\IResponseFactory;
use Psr\Log\LogLevel;

class UserModule implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        // Add a custom HTTP response for an exception
        $this->withExceptionResponseFactory(
            $appBuilder,
            UserNotFoundException::class,
            function (UserNotFoundException $ex, IHttpRequestMessage $request, IResponseFactory $responseFactory) {
                return $responseFactory->createResponse($request, HttpStatusCodes::HTTP_NOT_FOUND);
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

You can add your own custom components to application builders.  They typically have `with*()` methods to let you configure the component, and a `build()` method that actually finishes building the component.

> **Note:** Binders aren't dispatched until just before `build()` is called on the components.  This means you can't inject dependencies from binders into your components - they won't have been bound yet.  So, if you need any dependencies inside the `build()` method, use the DI container to resolve them.

Let's say you prefer to use Symfony's router, and want to be able to add routes from your modules.  The first thing you should do is add a binder for the router so that the DI container can resolve it:

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

Now, let's define a component that allows us to add routes.

```php
use Aphiria\Api\App;
use Aphiria\Application\IComponent;
use Aphiria\DependencyInjection\IContainer;
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
        $this->container->for(
            App::class,
            fn (IContainer $container) => $container->resolve(SymfonyRouterRequestHandler::class)
        );
    }

    // Our own method for adding routes
    public function withRoutes(string $name, Route $route): self
    {
        $this->routes[$name] = $route;

        return $this;
    }
}
```

All that's left is to register the component, and start using it.

```php
use Aphiria\Framework\Api\Builders\ApiApplicationBuilder;
use Symfony\Component\Routing\Route;

$appBuilder = new ApiApplicationBuilder($container);
$appBuilder->withComponent(new SymfonyRouterComponent($container));

// Now use it

$appBuilder->getComponent(SymfonyRouterComponent::class)
    ->withRoutes('GetUserById', new Route('users/{id}'));
```

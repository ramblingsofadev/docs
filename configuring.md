<h1 id="doc-title">Configuring</h1>

<section class="toc" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
   1. [Application Builders vs Bootstrappers](#application-builders-vs-bootstrappers)
2. [Application Builders](#application-builders)
   1. [Building API Apps](#building-api-apps)
   2. [Building Console Apps](#building-console-apps)
   3. [Configuring Bootstrappers](#configuring-bootstrappers)
   4. [Configuring Middleware](#configuring-middleware)
   5. [Configuring Console Commands](#configuring-console-commands)
3. [Components](#components)
4. [Modules](#modules)
5. [Using Aphiria Components](#using-aphiria-components)
   1. [Configuring Routes](#configuring-routes)
   2. [Configuring Encoders](#configuring-encoders)
   3. [Configuring Exception Log Levels](#configuring-exception-log-levels)
   4. [Configuring Exception Responses](#configuring-exception-responses)

</section>

<h2 id="basics">Basics</h2>

Aphiria comes with an easy way to centrally configure your application logic, eg bootstrappers, routes, console commands, and exception handlers.  It even lets you centralize the configuration of entire [modules](#modules) of code.

<h3 id="application-builders-vs-bootstrappers">Application Builders vs Bootstrappers</h3>

Before we dive too deep, you might be asking yourself "What's the difference between [application builders](#application-builders) and [bootstrappers](bootstrappers.md)?".  Bootstrappers are where you bind your dependencies to the DI container so that the container can resolve them - that's it.  Application builders, on the other hand, are where you can configure a [module](#modules) of your domain with shared resources, such as a [router](#configuring-routes) or even [bootstrappers](#configuring-bootstrappers) themselves.  This is really the heart of Aphiria - your app should be nothing but a collection of modules that are capable of configuring themselves.

<h2 id="application-builders">Application Builders</h2>

`ApplicationBuilder` provides all the functionality you'll need to configure your application logic:

```php
use Aphiria\Configuration\ApplicationBuilder;
use Aphiria\DependencyInjection\Bootstrappers\Inspections\BindingInspectorBootstrapperDispatcher;
use Aphiria\DependencyInjection\Container;

$container = new Container();
$bootstrapperDispatcher = new BindingInspectorBootstrapperDispatcher($container);
$appBuilder = new ApplicationBuilder($container, $bootstrapperDispatcher);
```

Once you're done configuring your [bootstrappers](#configuring-bootstrappers), [routes](#configuring-routes), and [console commands](#configuring-console-commands), you can go ahead and build your application.

<h3 id="building-api-apps">Building API Apps</h3>

Let's create an API app:

```php
$requestHandler = $appBuilder->buildApiApplication();
```

`$requestHandler` will be an instance of `IRequestHandler`.  Building your API will also dispatch bootstrappers and build your components.

<h3 id="building-console-apps">Building Console Apps</h3>

Let's create a console app:

```php
$kernel = $appBuilder->buildConsoleApplication();
```

`$kernel` will be an instance of `ICommandBus`.  Like [building an API app](#building-api-apps), your bootstrappers will also be dispatched and your components built.

<h3 id="configuring-bootstrappers">Configuring Bootstrappers</h3>

Let's take a look at how to configure bootstrappers:

```php
$appBuilder->withBootstrappers(fn () => [new FooBootstrapper]);
```

<h3 id="configuring-middleware">Configuring Middleware</h3>

You can configure your app to have global middleware, which doesn't have to be coupled to any particular middleware library implementation.  Simply set up a `MiddlewareBinding` with the class name and any attributes (optional):

```php
use Aphiria\Configuration\Middleware\MiddlewareBinding;

$appBuilder->withGlobalMiddleware(fn () => [
    new MiddlewareBinding(AuthMiddleware::class)
]);
```

Now, the `AuthMiddleware` will be run on every single request in your application.

<h3 id="configuring-console-commands">Configuring Console Commands</h3>

Now, we'll add some console commands:

```php
$appBuilder->withConsoleCommands(function (CommandRegistry $commands) {
    // You can also use $commands->registerManyCommands()
    $commands->registerCommand(
        new FooCommand(),
        fn () => new FooCommandHandler()
    );
});
```

<h2 id="components">Components</h2>

A component is a singular piece of your app, eg a [router](#configuring-routes), and is smaller in scope than a [module](#module).  Components are configured after bootstrappers are run, and are a convenient place to finish setting up your application before it runs.

To build a component in your app builder, you must first register a factory for it:

```php
$appBuilder->registerComponentBuilder('routes', function (array $callbacks) {
    // Assume we're using some generic routing library
    $router = new Router();

    // $callbacks will contain all callbacks registered to this component via withComponent()
    foreach ($callbacks as $callback) {
        $callback($router);
    }
});
```

Now, we can begin configuring this component.  For example, let's say we wanted to add some routes.  Just use the same component name as when we registered the component factory:

```php
$appBuilder->withComponent('routes', function (Router $router) {
    $router->get('/users', UserController::class, 'getUsers');
});
```

In this example, each callback will be invoked by the factory registered in `registerComponentBuilder`, and will be injected with an instance of `Router`.

For an added bit of syntactic sugar, you can just call `with{ComponentName}`:

```php
$appBuilder->withRoutes(function (Router $router) {
    $router->get('/users', UserController::class, 'getUsers');
});
```

This is semantically identical to our previous example that called `withComponent('routes', ...)`.

<h2 id="modules">Modules</h2>

A module is a chunk of your domain.  For example, if you are running a site where users can buy books, you might have a user module, a book module, and a shopping cart module.  Each of these modules will have separate bootstrappers, routes, and console commands.  So, why not bundle all the configuration logic by module?

```php
use Aphiria\Configuration\IModuleBuilder;
use Aphiria\Routing\Builders\RouteGroupOptions;

final class UserModuleBuilder implements IModuleBuilder
{
    public function build(IApplicationBuilder $appBuilder): void
    {
        $appBuilder->withBootstrappers(fn () => [new UserServiceBootstrapper]);

        $appBuilder->withComponent('routes', function (RouteBuilderRegistry $routes) {
            // Let's prefix all our routes with 'users'
            $routes->group(new RouteGroupOptions('users'), function (RouteBuilderRegistry $routes) {
                $routes->get('/:id')
                    ->toMethod(UserController::class, 'getUserById');
            });
        });
        
        $appBuilder->withComponent('commands', function (CommandRegistry $commands) {
            $commands->registerCommand(
                new RunUserReportCommand(),
                fn () => new RunUserReportCommandHandler()
            );
        });
    }
}
```

To use your module in your application builder, just call:

```php
$appBuilder->withModule(new UserModuleBuilder());
$requestHandler = $appBuilder->buildApiApplication();
```

Now, your entire user module is configured and ready to go.

<h2 id="using-aphiria-components">Using Aphiria Components</h2>

The configuration library isn't strictly tied to Aphiria's [routing](routing.md), [route annotation](routing.md#route-annotations), [console](console.md), [encoder](serialization.md), or [exception handling](http-exception-handling.md) libraries.  However, if you do decide to use them, we've simplified how you can configure them:

```php
use Aphiria\Configuration\AphiriaComponentBuilder;
use Aphiria\DependencyInjection\Container;

// Assume we already have an app builder
$container = new Container;
(new AphiriaComponentBuilder($container))
    ->withExceptionHandlers($appBuilder)
    ->withExceptionLogLevelFactories($appBuilder)
    ->withExceptionResponseFactories($appBuilder)
    ->withRoutingComponent($appBuilder)
    ->withRouteAnnotations($appBuilder)
    ->withEncoderComponent($appBuilder);

// Finish configuring your app...

$requestHandler = $appBuilder->buildApiApplication();
```

These methods will set up components for your [exception handlers](http-exception-handling.md), [routes](#configuring-routes), and [encoders](#configuring-encoders).

> **Note:** If you use Aphiria's exception handler library, it's highly recommended that you include it before building any other Aphiria components so that the exception handler middleware is registered first.

<h3 id="configuring-routes">Configuring Routes</h3>

You can manually register routes to your application:

```php
(new AphiriaComponentBuilder($container))
    ->withRoutingComponent($appBuilder);

// Then, inside your module:
$appBuilder->withComponent('routes', function (RouteBuilderRegistry $routes) {
    $routes->get('users/:id')
        ->toMethod(UserController::class, 'getUserById');
});
```

Due to how lazy route creation works, your routes will only be built if they need to be, eg they're not cached yet.

> **Note:** If you're using route annotations, those routes will be combined with any manually-registered routes.

<h3 id="configuring-encoders">Configuring Encoders</h3>

Sometimes our models require some custom encoding logic when serializing and deserializing them.  Let's configure an encoder for a user model:

```php
(new AphiriaComponentBuilder($container))
    ->withEncoderComponent($appBuilder);

// Then, inside your module:
$appBuilder->withComponent('encoders', function (EncoderRegistry $encoders) {
    $encoders->registerEncoder(User::class, new class() implements IEncoder {
        public function decode($userHash, string $type, EncodingContext $context)
        {
            return new User($userHash['id'], $userHash['email']);
        }

        public function encode($user, EncodingContext $context)
        {
            return ['id' => $user->getId(), 'email' => $user->getEmail()];
        }
    });
});
```

<h3 id="configuring-exception-log-levels">Configuring Exception Log Levels</h3>

Typically, uncaught exceptions get logged as `LogLevel::ERROR`.  However, there might be exceptions that warrant a higher or lower level.  For example, if you receive an exception that a database table is gone, you might want to log a `LogLevel::EMERGECNCY`.

```php
(new AphiriaComponentBuilder($container))
    ->withExceptionLogLevelFactories($appBuilder);

// Then, inside your module:
$appBuilder->withComponent(
    'exceptionLogLevelFactories', 
    function (ExceptionLogLevelFactoryRegistry $factories) {
        $factories->registerFactory(
            DbTableNotFoundException::class,
            fn (DbTableNotFoundException $ex) => LogLevel::EMERGENCY
        );
    }
);
```

<h3 id="configuring-exception-responses">Configuring Exception Responses</h3>

Aphiria has an easy way to map your module's exceptions to HTTP responses:

```php
(new AphiriaComponentBuilder($container))
    ->withExceptionResponseFactories($appBuilder);

// Then, inside your module:
$appBuilder->withComponent(
    'exceptionResponseFactories',
    function (ExceptionResponseFactoryRegistry $factories) {
        $factories->registerFactory(
            UserNotFoundException::class,
            fn (UserNotFoundException $ex, ?IHttpRequestMessage $request, INegotiatedResponseFactory $responseFactory) 
                => $responseFactory->createResponse($request, 404)
        );
    }
);
```

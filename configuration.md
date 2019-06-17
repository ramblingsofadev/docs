# Configuration

## Table of Contents
1. [Basics](#basics)
2. [Application Builders](#application-builders)
  1. [Configuring Bootstrappers](#configuring-bootstrappers)
  2. [Configuring Middleware](#configuring-middleware)
3. [Components](#components)
4. [Modules](#modules)
5. [Using Aphiria Components](#using-aphiria-components)
  1. [Configuring Routes](#configuring-routes)
  2. [Configuring Console Commands](#configuring-console-commands)

<h1 id="basics">Basics</h1>

Aphiria comes with an easy way to centrally configure your application logic, eg bootstrappers, routes, and console commands.  It even lets you centralize the configuration of entire [modules](#modules) of code.

<h1 id="application-builders">Application Builders</h1>

`ApplicationBuilder` provides all the functionality you'll need to configure your application logic:

```php
use Aphiria\Configuration\ApplicationBuilder;
use Opulence\Ioc\Bootstrappers\Inspections\BindingInspectorBootstrapperDispatcher;
use Opulence\Ioc\Container;

$container = new Container();
$bootstrapperDispatcher = new BindingInspectorBootstrapperDispatcher($container);
$appBuilder = new ApplicationBuilder($container, $bootstrapperDispatcher);
```

Once you're done configuring your [bootstrappers](#configuring-bootstrappers), [routes](#configuring-routes), and [console commands](#configuring-console-commands), you can go ahead and build your application:

```php
$appBuilder->build();
```

Your application logic is now all set to go.

<h2 id="configuring-bootstrappers">Configuring Bootstrappers</h2>

Let's take a look at how to configure bootstrappers:

```php
$appBuilder->withBootstrappers(fn () => [new FooBootstrapper]);
```

<h2 id="configuring-middleware">Configuring Middleware</h2>

TODO

<h1 id="components">Components</h1>

A component is a singular piece of your app, eg a [router](#configuring-routes), and is smaller in scope than a [module](#module).  Components are configured after bootstrappers are run, and are a convenient place to finish setting up your application before it runs.

To build a component in your app builder, you must first register a factory for it:

```php
$appBuilder->registerComponentFactory('routes', function (array $callbacks) {
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

In this example, each callback will be invoked by the factory registered in `registerComponentFactory`, and will be injected with an instance of `Router`.

For an added bit of syntactic sugar, you can just call `with{ComponentName}`:

```php
$appBuilder->withRoutes(function (Router $router) {
    $router->get('/users', UserController::class, 'getUsers');
});
```

This is semantically identical to our previous example that called `withComponent('routes', ...)`.

<h1 id="modules">Modules</h1>

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
                $routes->map('GET', '/:id')
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
$appBuilder->build();
```

Now, your entire user module is configured and ready to go.

<h1 id="using-aphiria-components">Using Aphiria Components</h1>

The configuration library isn't strictly tied to Aphiria's [routing](routing) or [console](console) libraries.  However, if you do decide to use them, we've simplified how you can configure them:

```php
use Aphiria\Configuraiton\AphiriaComponentBuilder;
use Opulence\Ioc\Container;

// Assume we already have an app builder
$container = new Container;
(new AphiriaComponentBuilder($container))
    ->withRoutingComponent($appBuilder)
    ->withCommandComponent($appBuilder);

// Finish configuring your app...

$app = $appBuilder->build();
```

These methods will set up components for your [routes](#configuring-routes) and [console commands](#configuring-console-commands).

<h2 id="configuring-routes">Configuring Routes</h2>

We can register some routes to your application:

```php
$appBuilder->withComponent('routes', function (RouteBuilderRegistry $routes) {
    $routes->map('GET', 'users/:id')
        ->toMethod(UserController::class, 'getUserById');
});
```

Due to how lazy route creation works, your routes will only be built if they need to be, eg they're not cached yet.

<h2 id="configuring-console-commands">Configuring Console Commands</h2>

Now, we'll add some console commands:

```php
$appBuilder->withComponent('commands', function (CommandRegistry $commands) {
    // You can also use $commands->registerManyCommands()
    $commands->registerCommand(
        new FooCommand(),
        fn () => new FooCommandHandler()
    );
});
```
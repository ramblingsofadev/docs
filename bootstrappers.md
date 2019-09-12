<h1 id="doc-title">Bootstrappers</h1>

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
2. [Inspection Bindings](#inspection-bindings)
3. [Using App Builders](#using-app-builders)

<h2 id="basics">Basics</h2>

A bootstrapper is a simple class that registers bindings to the container for a particular area of your domain.  For example, you might have a bootstrapper called `UserBootstrapper` to centralize all the bindings related to your user domain.

```php
use Aphiria\DependencyInjection\Bootstrappers\Bootstrapper;
use Aphiria\DependencyInjection\IContainer;

final class UserBootstrapper extends Bootstrapper
{
    public function registerBindings(IContainer $container): void
    {
        $container->bindInstance(IUserRepository::class, new UserRepository());
        $container->bindSingleton(IUserService::class, UserService::class);
    }
}
```

Prior to handling a request, you can dispatch all your dispatchers:

```php
$bootstrappers = [new UserBootstrapper];

foreach ($bootstrappers as $bootstrapper) {
    $bootstrapper->registerBindings($container);
}

// Your app is all set
```

Although this is simple, it's probably a little heavy-handed to register all your bindings for each request when only a few will be needed to handle a request.  The [next section](#inspection-bindings) will go into more details on how to handle this more optimally.

<h2 id="inspection-bindings">Inspection Bindings</h2>

Rather than having to dispatch _every_ bootstrapper on every request, you can use binding inspections to lazily register them, eg only when they're actually needed.  At a high level, we can look inside your bootstrappers and determine what each of them bind.  It can then set up a factory for each of those bindings that runs `Bootstrapper::registerBindings()` only when at least one of the bootstrapper's bindings is necessary.  This prevents you from having to list out the bindings a bootstrapper registers to get this sort of functionality (like other frameworks force you to do).

Let's build on the `UserBootstrapper` from the [previous example](#basics) and set up our app to inspect its bindings:

```php
use Aphiria\DependencyInjection\Bootstrappers\Inspection\BindingInspectorBootstrapperDispatcher;
use Aphiria\DependencyInjection\Bootstrappers\Inspection\Caching\FileBootstrapperBindingCache;
use Aphiria\DependencyInjection\Container;

$bootstrappers = [new UserBootstrapper()];
$container = new Container();
$bootstrapperDispatcher = new BindingInspectorBootstrapperDispatcher(
    $container,
    getenv('ENV_NAME') === 'production'
        ? new FileBootstrapperBindingCache('/tmp/bootstrapperInspections.txt')
        : null
);
$bootstrapperDispatcher->dispatch($bootstrappers);
```

That's it.  Now, whenever we call `$container->resolve(IUserService::class)`, it will automatically run `UserBootstrapper::registerBindings()` once and use the binding defined inside to resolve it every time after.

> **Note:** It's recommended that you only use caching for bootstrapper bindings in production environments.  Otherwise, changes you make to your bootstrappers might not be reflected.

<h2 id="using-app-builders">Using App Builders</h2>

The [configuration](configuring.md) library provides helper methods to simplify building your application, including registering your bootstrappers.  Refer to [its documentation](configuring.md#configuring-bootstrappers) to learn more about it.

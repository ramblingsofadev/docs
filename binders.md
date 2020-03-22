<h1 id="doc-title">Binders</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
2. [Inspection Bindings](#inspection-bindings)
3. [Using App Builders](#using-app-builders)

</div>

</nav>

<h2 id="basics">Basics</h2>

A binder is a simple class that registers bindings to the container for a particular area of your domain.  For example, you might have a binder called `UserBinder` to centralize all the bindings related to your user domain.

```php
use Aphiria\DependencyInjection\Binders\Binder;
use Aphiria\DependencyInjection\IContainer;

final class UserBinder extends Binder
{
    public function bind(IContainer $container): void
    {
        $container->bindInstance(IUserRepository::class, new UserRepository());
        $container->bindSingleton(IUserService::class, UserService::class);
    }
}
```

Prior to handling a request, you can dispatch all your dispatchers:

```php
$binders = [new UserBinder];

foreach ($binders as $binder) {
    $binder->bind($container);
}

// Your app is all set
```

Although this is simple, it's probably a little heavy-handed to register all your bindings for each request when only a few will be needed to handle a request.  The [next section](#inspection-bindings) will go into more details on how to handle this more optimally.

<h2 id="inspection-bindings">Inspection Bindings</h2>

Rather than having to dispatch _every_ binder on every request, you can use binding inspections to lazily register them, eg only when they're actually needed.  At a high level, we can look inside your binders and determine what each of them bind.  It can then set up a factory for each of those bindings that runs `Binder::bind()` only when at least one of the binder's bindings is used.  This prevents you from having to list out the bindings a binder registers to get this sort of functionality (like other frameworks force you to do).

Let's build on the `UserBinder` from the [previous example](#basics) and set up our app to inspect its bindings:

```php
use Aphiria\DependencyInjection\Binders\Inspection\BindingInspectorBinderDispatcher;
use Aphiria\DependencyInjection\Binders\Inspection\Caching\FileBinderBindingCache;
use Aphiria\DependencyInjection\Container;

$binders = [new UserBinder()];
$container = new Container();
$binderDispatcher = new BindingInspectorBinderDispatcher(
    $container,
    getenv('ENV_NAME') === 'production'
        ? new FileBinderBindingCache('/tmp/binderInspections.txt')
        : null
);
$binderDispatcher->dispatch($binders);
```

That's it.  Now, whenever we call `$container->resolve(IUserService::class)`, it will automatically run `UserBinder::bind()` once and use the binding defined inside to resolve it every time after.

> **Note:** It's recommended that you only use caching for binder bindings in production environments.  Otherwise, changes you make to your binders might not be reflected.

<h2 id="using-app-builders">Using App Builders</h2>

The [configuration](application-builders.md) library provides helper methods to simplify building your application, including registering your binders.  Refer to [its documentation](application-builders.md#configuring-binders) to learn more about it.

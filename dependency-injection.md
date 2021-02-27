<h1 id="doc-title">Dependency Injection</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
2. [Bindings](#bindings)
3. [Auto-Wiring](#auto-wiring)
   1. [Targeted Bindings](#targeted-bindings)
4. [Binders](#binders)
   1. [Dispatching Binders](#dispatching-binders)

</div>

</nav>

<h2 id="basics">Basics</h2>

Dependency injection (DI) is the process of passing (injecting) dependencies that a class needs, typically done via the constructor.  Benefits to DI include:

* Better testability - you can mock dependencies and inject them in unit tests
* Easier to detect when a class has too many dependencies - the constructor becomes bloated with parameters
* Cleaner abstraction in your interfaces - they don't leak the dependencies they need

A common way of defining what dependencies to inject for a particular class is using a DI container.  You can tell the container "When I need to use a particular interface, use this implementation of that interface".  Here's an example:

```php
use Aphiria\DependencyInjection\Container;

$container = new Container();
$container->bindInstance(IUserService::class, new UserService());

// Will return the same instance that was bound above
$userService = $container->resolve(IUserService::class);
```

<h2 id="bindings">Bindings</h2>

A binding is a way of telling the container what instance to use when resolving an interface.  There are a few different ways of registering bindings:

```php
// Whenever you need IUserService, always use the same instance of UserService
$container->bindInstance(IUserService::class, new UserService());

// Whenever you need IUserService, run the factory to get a new instance
$container->bindFactory(IUserService::class, fn () => new UserService());

// Whenever you need IUserService, run the factory and use that instance every time after
$container->bindFactory(IUserService::class, fn () => new UserService(), true);

// Whenever you need IUserService, use auto-wiring to return a new instance of UserService
$container->bindClass(IUserService::class, UserService::class);

// Whenever you need IUserService, use auto-wiring to return the same instance of UserService
$container->bindClass(IUserService::class, UserService::class, resolveAsSingleton: true);
```

> **Note:** The factory in `bindFactory()` **must** be parameterless.  Also, if you attempt to resolve an interface, but the container does not have a binding or it cannot [auto-wire](#auto-wiring) it, a `ResolutionException` will be thrown.

You can check if the container has a particular binding:

```php
if ($container->hasBinding(IUserService::class)) {
    // ...
}
```

You can also remove a binding:

```php
$container->unbind(IUserService::class);
```

If you'd like to try resolving a binding without an exception being thrown, use `tryResolve()`:

```php
$userService = null;

if (!$container->tryResolve(IUserService::class, $userService)) {
    $userService = new UserService();
}

// ...
```

<h2 id="auto-wiring">Auto-Wiring</h2>

Auto-wiring is when you let the container use reflection to scan the constructor and attempt to automatically instantiate each parameter.  Let's build off the [targeted binding example](#targeted-bindings).

```php
$container->bindInstance(IUserRepository::class, new UserRepository());
$userService = $container->resolve(UserService::class);
```

The container will scan `UserService::__construct()`, see the `IUserRepository` parameter, and check to see if it has a binding.  If it doesn't, it will attempt to auto-wire an instance of it if possible.

> **Note:** A parameter with `mixed` type cannot be auto-wired as anything other than a primitive value.  Also, the container will attempt to auto-wire union types, eg `string|Closure`, using the left-most type first, and only on failure will the container attempt to use the next left-most type.

<h3 id="targeted-bindings">Targeted Bindings</h3>

If you want a binding to only apply when auto-wiring a specific class, use a targeted binding.  Let's say that `UserService` looked like this:

```php
final class UserService implements IUserService
{
    public function __construct(private IUserRepository $userRepository) {}

    // ...
}
```

You can tell the container to use a specific instance of `IUserRepository` when resolving `UserService`:

```php
$container->for(
    UserService::class,
    fn ($container) => $container->bindInstance(IUserRepository::class, new UserRepository())
);
```

Calling `$container->resolve(UserService::class)` will now automatically inject an instance of `UserRepository`.

<h2 id="binders">Binders</h2>

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

If you're using the <a href="https://github.com/aphiria/app/issues" target="_blank">skeleton app</a>, you can register `UserBinder` to your app with an [application builder](configuration.md#component-binders).

<h3 id="dispatching-binders">Dispatching Binders</h3>

Aphiria does something unique - it automatically dispatches a binder only when one or more of its bindings are needed by your application.  It does this by constructing a graph between binders and bound/resolved interfaces, allowing it to dispatch the bare minimum number of binders to handle a request, increasing performance.

If you're using the <a href="https://github.com/aphiria/app/issues" target="_blank">skeleton app</a>, your binders will automatically be dispatched, and you can skip the rest of this section.  Otherwise, you'll have to manually dispatch your binders.  Rather than having to dispatch _every_ binder on every request, you can use `LazyBinderDispatcher` to lazily dispatch them, ie only when they're actually needed.  Let's build on the `UserBinder` from the [previous example](#binders) and set up our app to lazily dispatch it:

```php
use Aphiria\DependencyInjection\Binders\LazyBinderDispatcher;
use Aphiria\DependencyInjection\Binders\Metadata\Caching\FileBinderMetadataCollectionCache;
use Aphiria\DependencyInjection\Container;

$container = new Container();
$metadataCache = \getenv('ENV_NAME') === 'production'
    ? new FileBinderMetadataCollectionCache('/tmp/binderMetadataCollectionCache.txt')
    : null;
$binderDispatcher = new LazyBinderDispatcher($metadataCache);
$binderDispatcher->dispatch([new UserBinder()], $container);
```

That's it.  The first time we dispatch the binders, the binder metadata will be collected and cached for future requests.  Also, whenever a binder resolves an interface bound in another binder, that other binder will be automatically dispatched, too.

> **Note:** It's recommended that you only use caching for production environments.  Otherwise, changes you make to your binders might not be reflected.

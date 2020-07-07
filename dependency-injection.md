<h1 id="doc-title">Dependency Injection</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
2. [Bindings](#bindings)
3. [Targeted Bindings](#targeted-bindings)
4. [Auto-Wiring](#auto-wiring)
5. [Binders](#binders)
   1. [Lazy Dispatching](#lazy-dispatching)
   2. [Using Application Builders](#using-application-builders)

</div>

</nav>

<h2 id="basics">Basics</h2>

A dependency injection (DI) container gives you a way of telling your app "When you need an instance of `IFoo`, use this implementation".

```php
use Aphiria\DependencyInjection\Container;
use App\UserService;
use App\IUserService;

$container = new Container();
$container->bindInstance(IUserService::class, new UserService());

// ...

// Will be an instance of UserService
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
$container->bindClass(IUserService::class, UserService::class, [], true);
```

> **Note:** The factory in `bindFactory()` **must** be parameterless.

> **Note:** If you attempt to resolve an interface, but the container does not have a binding or it cannot [auto-wire](#auto-wiring) it, a `ResolutionException` will be thrown.

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
    $userService = new UserService(new UserRepository());
}

// ...
```

<h2 id="targeted-bindings">Targeted Bindings</h2>

If you only want to use a specific binding when resolving a type, you can use what's called a targeted binding.  Let's say that `UserService` looked like this:

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

<h2 id="auto-wiring">Auto-Wiring</h2>

Auto-wiring is when you let the container use reflection to scan the constructor and attempt to automatically instantiate each parameter.  Let's build off of the [targeted binding example](#targeted-bindings).

```php
$container->bindInstance(IUserRepository::class, new UserRepository());
$userService = $container->resolve(UserService::class);
```

The container will scan `UserService::__construct()`, see the `IUserRepository` parameter, and either check to see if it has a binding or attempt to auto-wire an instance of it if possible.

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

Prior to handling a request, you can dispatch all your binders:

```php
$binders = [new UserBinder()];

foreach ($binders as $binder) {
    $binder->bind($container);
}

// Your app is all set
```

Although this is simple, it's probably a little heavy-handed to register all your bindings for each request when only a few will be needed to handle a request.  The [next section](#lazy-dispatching) will go into more details on how to handle this more optimally.

<h3 id="lazy-dispatching">Lazy Dispatching</h3>

Rather than having to dispatch _every_ binder on every request, you can use `LazyBinderDispatcher` to lazily dispatch them, eg only when they're actually needed.  At a high level, it looks inside your binders to determine what each of them bind and resolve.  It then sets up a factory for each of those bindings that calls `Binder::bind()` only when at least one of the binder's bindings is used.  This prevents you from having to list out the bindings a binder registers to get this sort of functionality (like other frameworks force you to do).

Let's build on the `UserBinder` from the [previous example](#binders) and set up our app to lazily dispatch it:

```php
use Aphiria\DependencyInjection\Binders\LazyBinderDispatcher;
use Aphiria\DependencyInjection\Binders\Metadata\Caching\FileBinderMetadataCollectionCache;
use Aphiria\DependencyInjection\Container;

$container = new Container();
$metadataCache = getenv('ENV_NAME') === 'production'
    ? new FileBinderMetadataCollectionCache('/tmp/binderMetadataCollectionCache.txt')
    : null;
$binderDispatcher = new LazyBinderDispatcher($metadataCache);
$binderDispatcher->dispatch([new UserBinder], $container);
```

That's it.  Now, whenever we call `$container->resolve(IUserService::class)`, it will automatically run `UserBinder::bind()` once and use the binding defined inside to resolve it every time after.  Additionally, any binder that resolves interfaces bound by `UserBinder` will also be dispatched.

> **Note:** It's recommended that you only use caching for production environments.  Otherwise, changes you make to your binders might not be reflected.

<h3 id="using-application-builders">Using Application Builders</h3>

The [application builder](configuration.md#application-builders) library provides helper methods to simplify building your application, including registering your binders.  Refer to [its documentation](configuration.md#component-binders) to learn more about it.

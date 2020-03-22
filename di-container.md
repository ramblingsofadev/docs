<h1 id="doc-title">DI Container</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
2. [Bindings](#bindings)
3. [Targeted Bindings](#targeted-bindings)
4. [Auto-Wiring](#auto-wiring)
5. [Binders](#binders)

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
```

> **Note:** The factory **must** be parameterless.

```php
// Whenever you need IUserService, use auto-wiring to return the same instance of UserService
$container->bindSingleton(IUserService::class, UserService::class);

// Whenever you need IUserService, use auto-wiring to return a new instance of UserService
$container->bindPrototype(IUserService::class, UserService::class);
```

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
    private IUserRepository $userRepository;
    
    public function __construct(IUserRepository $userRepository)
    {
        $this->userRepository = $userRepository;
    }

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

For more info on binders, [read their documentation](binders.md).

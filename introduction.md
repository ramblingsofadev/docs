<h1 id="doc-title">Introduction</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [What Is It?](#what-is-it)
2. [Another PHP Framework?](#another-php-framework)
3. [Getting Started](#getting-started)

</div>

</nav>

<h2 id="what-is-it">What Is It?</h2>

Aphiria is a suite of decoupled libraries that help developers write expressive REST APIs without bleeding into your application.  For example, need an endpoint to create a user object?  Simple.

```php
final class UserController extends Controller
{
    // Assume this is from your domain
    private IUserService $userService;

    public function __construct(IUserService $userService)
    {
        $this->userService = $userService;
    }

    public function createUser(User $user): User
    {
        return $this->userService->createUser($user);
    }
}
```

Notice how our controller method takes in a `User` object, and also returns one?  What separates Aphiria from other frameworks is that it can perform [content negotiation](content-negotiation.md) on your plain old PHP objects (POPOs) so that you can write more expressive controllers and leave the messy parts of HTTP to Aphiria.

Let's actually configure our app to include this endpoint.  Let's say we need a [bootstrapper](bootstrappers.md) so that an instance of `IUserService` can be injected into the controller.  Easy.

```php
$appBuilder->withBootstrappers(fn () => [new UserServiceBootstrapper]);
$appBuilder->withRoutes(function (RouteBuilderRegistry $routes) {
    $routes->post('users')
        ->toMethod(UserController::class, 'createUser');
});
```

Our [application builder](configuring.md) simplifies registering all [bootstrappers](bootstrappers.md), [routes](routing.md), and [console commands](console.md) in an [area of your domain](configuring.md#modules).

<h2 id="another-php-framework">Another PHP Framework?</h2>

Great question.  The idea for Aphiria was conceived after using ASP.NET Core.  Its expressive syntax, intuitive models, and simple configuration inspired me to see if I could find these things in a PHP framework.  I looked at frameworks (even Opulence), and usually found at least one major problem with them all:
 
* Lack of expressive controllers that supported [content negotiation](content-negotiation.md)
* Too much magic going on behind the scenes
* All-in adoption of some of the less popular PSRs

I spent months sketching out the ideal syntax.  As I waded into the depths of development, though, I started to realize that my ideas were a fundamental shift from Opulence 1.0.  For example:
  
* To make content negotiation work, I'd need to completely rewrite my abstractions for HTTP requests and responses
* I'd need a whole library dedicated to the serializing and deserializing of POPOs with no/minimal configuration
* The Opulence router would need to be rewritten to keep up with the speed of the latest routing libraries

After more than a year of development, a whole new framework was emerging - Aphiria.  I honestly believe it is the most intuitive, expressive PHP framework to date.  It enables developers to build powerful APIs without getting in the way.  I invite you to browse [some of the documentation](controllers.md) and see for yourself.

<h2 id="getting-started">Getting Started</h2>

To get up and running, follow our [installation guide](installation.md) to create a skeleton app that uses Aphiria.  Then, [learn how to define some routes](routing.md),  [create some controllers](controllers.md), and [configure your dependencies](bootstrappers.md).  From there, you can browse the docs in any order you choose, although the order they're listed in might be the best way to read them.

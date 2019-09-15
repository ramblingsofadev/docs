<h1 id="doc-title">Introduction</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [What Is It?](#what-is-it)
2. [Getting Started](#getting-started)
3. [Another PHP Framework?](#another-php-framework)

</div>

</nav>

<h2 id="what-is-it">What Is It?</h2>

Aphiria is a suite of decoupled libraries that help you write expressive REST APIs without bleeding into your code base.  Let's look at an example controller.

```php
final class UserController extends Controller
{
    // Assume this is from your domain
    private IUserService $users;

    public function __construct(IUserService $users)
    {
        $this->users = $users;
    }

    public function createUser(CreateUserRequest $createUserRequest): IHttpResponseMessage
    {
        $user = $this->users->createUser($createUserRequest->email, $createUserRequest->password);
        
        return $this->created("users/{$user->id}", $user);
    }

    public function getUser(int $id): User
    {
        return $this->users->getUserById($id);
    }
}
```

In `createUser()`, Aphiria uses [content negotiation](content-negotiation.md) to deserialize the request body to a `CreateUserRequest`.  Likewise, Aphiria determines how to serialize the user in `getUser()` to whatever format the client wants (eg JSON).  This is all done with zero configuration of your plain-old PHP objects (POPOs).

Now, we'll actually set up our app to include these endpoints.  Let's say we need a [bootstrapper](bootstrappers.md) so that an instance of `IUserService` can be injected into the controller.  Easy.

```php
$appBuilder->withBootstrappers(fn () => [new UserServiceBootstrapper]);
$appBuilder->withRoutes(function (RouteBuilderRegistry $routes) {
    $routes->post('users')
        ->toMethod(UserController::class, 'createUser');

    $routes->get('users/:id')
        ->toMethod(UserController::class, 'getUser');
});
```

The [application builder](application-builders.md) simplifies registering all [bootstrappers](application-builders.md#configuring-bootstrappers), [routes](application-builders.md#configuring-routes), and other parts of your application in a [modular way](application-builders.md#modules).

Hopefully, these examples demonstrate how easy it is to build an application with Aphiria.

<h2 id="getting-started">Getting Started</h2>

To get up and running, follow our [installation guide](installation.md) to create a skeleton app that uses Aphiria.  Then, [learn how to define some routes](routing.md),  [create some controllers](controllers.md), and [configure your dependencies](bootstrappers.md).  From there, you can browse the docs in any order you choose, although the order they're listed in might be the best way to read them.

<h2 id="another-php-framework">Another PHP Framework?</h2>

Great question.  The idea for Aphiria was conceived after using ASP.NET Core.  Its expressive syntax, intuitive models, and simple configuration inspired me to see if I could find these things in a PHP framework.  I looked at frameworks (even <a href="https://www.opulencephp.com" target="_blank" title="Opulence">Opulence</a>), and usually found at least one major problem with them all:
 
* Lack of support for [content negotiation](content-negotiation.md)
* Too much magic going on behind the scenes
* All-in adoption of some of the less popular PSRs

I spent months sketching out the ideal syntax.  As I waded into the depths of development, though, I started to realize that my ideas were a fundamental shift from Opulence.  For example:
  
* To make content negotiation work, I'd need to completely rewrite my abstractions for HTTP requests and responses
* I'd need a whole library dedicated to the serializing and deserializing of POPOs with no/minimal configuration
* The Opulence router would need to be rewritten to keep up with the speed of the latest routing libraries

After more than a year of development, a whole new framework was emerging - Aphiria.  I honestly believe it is the most intuitive, expressive PHP framework to date.  It enables developers to build powerful APIs without getting in the way.  I invite you to browse [some of the documentation](controllers.md) and see for yourself.

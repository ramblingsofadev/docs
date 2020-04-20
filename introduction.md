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
    private IUserService $users;

    public function __construct(IUserService $users)
    {
        $this->users = $users;
    }

    public function createUser(User $user): IResponse
    {
        $createdUser = $this->users->createUser($user->email, $user->password);

        return $this->created("users/{$createdUser->id}", $createdUser);
    }

    public function getUser(int $id): User
    {
        return $this->users->getUserById($id);
    }
}
```

In `createUser()`, Aphiria uses [content negotiation](content-negotiation.md) to deserialize the request body to a `User`.  Likewise, Aphiria determines how to serialize the user in `getUser()` to whatever format the client wants (eg JSON).  This is all done with zero configuration of your plain-old PHP objects (POPOs).

Now, we'll actually set up our app to include these endpoints.  Let's say we need a [binder](binders.md) so that an instance of `IUserService` can be injected into the controller.  Easy.

```php
$appBuilder->withBinders(fn () => [new UserServiceBinder]);
$appBuilder->withRoutes(function (RouteCollectionBuilder $routes) {
    $routes->post('users')
        ->mapsToMethod(UserController::class, 'createUser');

    $routes->get('users/:id')
        ->mapsToMethod(UserController::class, 'getUser');
});
```

The [application builder](application-builders.md) simplifies registering all [binders](application-builders.md#component-binders), [routes](application-builders.md#component-routes), and other parts of your application in a [modular way](application-builders.md#basics).

Hopefully, these examples demonstrate how easy it is to build an application with Aphiria.

<h2 id="getting-started">Getting Started</h2>

To get up and running, follow our [installation guide](installation.md) to create a skeleton app that uses Aphiria.  Then, [learn how to define some routes](routing.md),  [create some controllers](controllers.md), and [configure your dependencies](binders.md).  From there, you can browse the docs in any order you choose, although the order they're listed in might be the best way to read them.

<h2 id="another-php-framework">Another PHP Framework?</h2>

Great question.  The idea for Aphiria was conceived after using ASP.NET Core.  Its expressive syntax, intuitive models, and simple configuration inspired me to see if I could find these things in a PHP framework.  I looked at frameworks (even <a href="https://www.opulencephp.com" target="_blank" title="Opulence">Opulence</a>), and usually found at least one major problem with them all:
 
* A lot of coupling between framework libraries, making it difficult to substitute in third party libraries
* Lack of support for [automatic content negotiation](content-negotiation.md)
* Lack of simple, [code-based application configuration](application-builders.md)
* No [code-based model validators](validation.md)
* No baked-in, optional support for [route](routing.md#route-annotations), [command](console.md#command-annotations), or [validator](validation.md#validation-annotations) annotations
* Generally too much magic going on behind the scenes
* All-in adoption of some less popular PSRs

There is a general trend towards having separate API and front end codebases.  Full stack frameworks have their place, but I felt like this was becoming a less and less common way to write websites - most are JavaScript-powered UIs calling APIs.  So, I focused on what I know best - building REST APIs.  I spent months sketching out the ideal syntax, laboring over how to provide the best developer experience possible.  Once I had the syntax down, I worked backwards and started implementing it over the course of a few years.  It wasn't easy, but I can honestly say I am very proud of it.  I hope you enjoy it, too.

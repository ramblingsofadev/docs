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
    public function __construct(private IUserService $users) {}

    public function createUser(Credentials $creds): IResponse
    {
        $user = $this->users->createUser($creds->email, $creds->password);

        return $this->created("/users/{$user->id}", $user);
    }

    public function getUser(int $id): User
    {
        return $this->users->getUserById($id);
    }
}
```

In `createUser()`, Aphiria uses [content negotiation](content-negotiation.md) to deserialize the request body to a `Credentials` object.  Likewise, Aphiria determines how to serialize the `User` from `getUser()` to whatever format the client wants (eg JSON or XML).  This is all done with zero configuration of your plain-old PHP objects (POPOs).

To use these endpoints, we'll need to include a [binder](dependency-injection.md#binders) so that an instance of `IUserService` can be injected into the controller, and we need to define our routes.  You can centrally configure all of these parts of your user domain in a [module](configuration.md#modules).

```php
final class UserModule implement IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        $this->withBinders($appBuilder, fn () => [new UserServiceBinder])
            ->withRoutes($appBuilder, function (RouteCollectionBuilder $routes) {
                $routes->post('/users')
                    ->mapsToMethod(UserController::class, 'createUser');
            
                $routes->get('/users/:id')
                    ->mapsToMethod(UserController::class, 'getUser');
            });
    }
}
```

Modules allow you to configure each of your business domains individually, making it trivial to plug-and-play new ones.  For example, you can specify [binders](configuration.md#component-binders), [routes](configuration.md#component-routes), [validators](configuration.md#component-validator), and other components that your domain needs.

Hopefully, these examples demonstrate how easy it is to build an application with Aphiria.

<h2 id="getting-started">Getting Started</h2>

To get up and running, follow our [installation guide](installation.md) to create a skeleton app that uses Aphiria.  Then, [learn how to define some routes](routing.md),  [create some controllers](controllers.md), and [configure your dependencies](dependency-injection.md#binders).  From there, you can browse the docs in any order you choose, although the order they're listed in might be the best way to read them.

Aphiria uses a GitHub project for keeping track of new features, bug fixes, and roadmapping.  To learn more about the direction of the framework, check out the <a href="https://github.com/orgs/aphiria/projects/1" target="_blank">project</a>.

<h2 id="another-php-framework">Another PHP Framework?</h2>

Great question.  The idea for Aphiria was conceived after using ASP.NET Core.  Its expressive syntax, intuitive models, and simple configuration inspired me to see if I could find these things in a PHP framework.  I looked at frameworks, and usually found at least one major problem with them all:
 
* A lot of coupling between framework libraries, making it difficult to substitute in third party libraries
* Lack of support for [automatic content negotiation](content-negotiation.md)
* Lack of simple, [code-based application configuration](configuration.md#application-builders)
* No [code-based model validators](validation.md)
* No baked-in, optional support for [route](routing.md#route-attributes), [command](console.md#command-attributes), or [validator](validation.md#validation-attributes) attributes
* Generally too much magic going on behind the scenes
* All-in adoption of some less popular PSRs

There is a general trend towards having separate API and front end codebases.  Full stack frameworks have their place, but I felt like this was becoming a less and less common way to write websites - most are JavaScript-powered UIs calling APIs.  So, I focused on what I know best - building REST APIs.  I spent months sketching out the ideal syntax, laboring over how to provide the best developer experience possible.  Once I had the syntax down, I worked backwards and started implementing it over the course of a few years.  It wasn't easy, but I can honestly say I am very proud of it.  I hope you enjoy it, too.

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

    #[Post('/users')]
    public function createUser(Credentials $creds): IResponse
    {
        $user = $this->users->createUser($creds->email, $creds->password);

        return $this->created("/users/{$user->id}", $user);
    }

    #[Get('/users/:id')]
    public function getUser(int $id): User
    {
        return $this->users->getUserById($id);
    }
}
```

In `createUser()`, Aphiria uses [content negotiation](content-negotiation.md) to deserialize the request body to a `Credentials` object.  Likewise, Aphiria determines how to serialize the `User` from `getUser()` to whatever format the client wants (eg JSON or XML).  This is all done with **zero configuration** of your plain-old PHP objects (POPOs).

Let's configure our [DI container](dependency-injection.md) to inject an instance of `IUserService` into our controller via a [binder](dependency-injection.md#binders).

```php
final class UserServiceBinder extends Binder
{
    public function bind(IContainer $container): void
    {
        $container->bindInstance(IUserService::class, new UserService());
    }
}
```

Next, let's use a [module](configuration.md#modules) to register the binder.  We'll also configure our app to map an exception that `IUserService` might throw to an HTTP response.  Modules give you a place to configure each piece of your business domain, allowing you to easily plug-and-play code into your app.

```php
final class UserModule implements IModule
{
    use AphiriaComponents;

    public function configure(IApplicationBuilder $appBuilder): void
    {
        $this->withBinders($appBuilder, new UserServiceBinder())
            ->withProblemDetails(
                $appBuilder,
                UserNotFoundException::class,
                status: HttpStatusCodes::NOT_FOUND
            );
    }
}
```

Finally, let's register the module with our app, which is itself a module.

```php
final class GlobalModule implements IModule
{
    use AphiriaComponents;
    
    public function configure(IApplicationBuilder $appBuilder): void
    {
        $this->withModules($appBuilder, new UserModule());
    }
}
```

That's all it takes to build a fully-functional API with Aphiria.

<h2 id="getting-started">Getting Started</h2>

To get up and running, follow our [installation guide](installation.md) to create a skeleton app that uses Aphiria.  Then, [learn how to define some routes](routing.md),  [create some controllers](controllers.md), and [configure your dependencies](dependency-injection.md#binders).  From there, you can browse the docs in any order you choose, although the order they're listed in might be the best way to read them.

Aphiria uses a GitHub project for keeping track of new features, bug fixes, and roadmapping.  To learn more about the direction of the framework, check out the <a href="https://github.com/orgs/aphiria/projects/1" target="_blank">project</a>.

<h2 id="another-php-framework">Another PHP Framework?</h2>

Great question.  The idea for Aphiria was conceived after using ASP.NET Core.  Its expressive syntax, intuitive models, and simple configuration inspired us to see if we could find these things in a PHP framework.  We looked at frameworks, and usually found at least one major problem with them all:
 
* A lot of coupling between framework libraries, making it difficult to substitute in third party libraries
* Lack of support for [automatic content negotiation](content-negotiation.md)
* Lack of simple, [code-based application configuration](configuration.md#application-builders)
* No [code-based model validators](validation.md)
* No baked-in, optional support for [route](routing.md#route-attributes), [command](console.md#command-attributes), or [validator](validation.md#creating-a-validator) attributes
* Generally too much magic going on behind the scenes
* All-in adoption of some less popular PSRs

There is a general trend towards having separate API and front end codebases.  Full stack frameworks have their place, but we felt like this was becoming a less and less common way to write websites - most are JavaScript-powered UIs calling APIs.  So, we focused on what we know best - building REST APIs.  We spent months sketching out the ideal syntax, laboring over how to provide the best developer experience possible.  Once we had the syntax down, we worked backwards and started implementing it over the course of a few years.  It wasn't easy, but we can honestly say we are very proud of it.  We hope you enjoy it, too.

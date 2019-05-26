# Introduction

## Table of Contents
1. [Basics](#basics)
2. [PSRs](#psrs)

<h1 id="basics">Basics</h1>

Aphiria is a simple, expressive PHP framework to build APIs with.  It's comprised of several decoupled libraries, and also builds on top of some libraries from <a href="https://www.opulencephp.com" target="_blank">Opulence</a>.  For example, need an endpoint to create a user object?  Simple:

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

Then, map a route to that endpoint:

```php
$routes->map('POST', '')
    ->toMethod(UserController::class, 'createUser');
```

Notice how our controller method takes in a `User` object, and also returns one?  What separates Aphiria from other frameworks is that it can perform [content negotiation](content-negotiation) on your plain old PHP objects (POPOs) so that you can write more expressive controllers and leave the messy parts of HTTP to Aphiria.

<h1 id="psrs">PSRs</h1>

The FIG did a great job with some of their earlier PHP Standards Recommendations (PSRs), but PSRs like PSR-7 were controversial for reasons like their decision to go with immutability.  Although it was a noble attempt, Aphiria attempts to fix some of the issues with the later PSRs, eg:

* [An improved API for HTTP models](net)
* [Content negotiation](content-negotiation)
* [A more expressive DI container](di-container)

The decision to not embrace all the PSRs is certainly a controversial one, and we understand not everyone will be happy with it.  However, we hope after you read through Aphiria's alternatives, you'll be convinced of their superiority.
<h1 id="doc-title">Authorization</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Introduction](#introduction)
2. [Policies](#policies)
3. [Configuring an Authority](#configuring-an-authority)
4. [Resource Authorization](#resource-authorization)
5. [Authorization Results](#authorization-results)
6. [Customizing Failed Authorization Responses](#customizing-failed-authorization-responses)

</div>

</nav>

<h2 id="introduction">Introduction</h2>

Authorization is the act of checking whether or not an action is permitted, and typically goes hand-in-hand with [authentication](authentication.md).  Aphiria provides policy-based authorization.  A policy is a definition of requirements that must be passed for authorization to succeed.  One such example of a requirement is whether or not a user has a particular role.

Let's look at an example of role authorization using attributes:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Authorization\Attributes\AuthorizeRoles;
use Aphiria\Net\Http\IResponse;
use Aphiria\Routing\Attributes\Post;

final class ArticleController extends Controller
{
    #[Post('/article')]
    #[AuthorizeRoles(['admin', 'contributor', 'editor'])]
    public function createArticle(Article $article): IResponse
    {
        // ...
    }
}
```

Here's the identical functionality, just using `IAuthority` instead of an attribute:

```php
use Aphiria\Authentication\IUserAccessor;
use Aphiria\Authorization\IAuthority;
use Aphiria\Authorization\AuthorizationPolicy;
use Aphiria\Authorization\RequirementHandlers\RolesRequirement;

final class ArticleController extends Controller
{
    public function __construct(private IAuthority $authority, private IUserAccessor $userAccessor) {}

    #[Post('/article')]
    public function createArticle(Article $article): IResponse
    {
        $user = $this->userAccessor->getUser($this->request);
        $policy = new AuthorizationPolicy(
            'create-article',
            new RolesRequirement(['admin', 'contributor', 'editor'])
        );
    
        if (!$this->authority->authorize($user, $policy)->passed) {
            return $this->forbidden();
        }
        
        // ...
    }
}
```

> **Note:** `IAuthority::authorize()` accepts both a policy name and an `AuthorizationPolicy` instance.

Authorization isn't just limited to checking roles.  In the next section, we'll discuss how policies can be used to authorize against many different types of data.

<h2 id="policies">Policies</h2>

A policy consists of a name, one or more requirements, and the [authentication scheme](authentication.md#authentication-schemes) to use.  A policy can check whether or not a principal's [claims](authentication.md#claims) pass the requirements.

Let's say our application requires users to be at least 13 years old to use it.  In this case, we'll create a policy that checks the `ClaimType::DateOfBirth` claim.  First, let's define our POPO requirement class:

```php
final class MinimumAgeRequirement
{
    public function __construct(public readonly int $minimumAge) {}
}
```

Next, let's create a handler that checks this requirement:

```php
use Aphiria\Authorization\AuthorizationContext;
use Aphiria\Authorization\IAuthorizationRequirementHandler;
use Aphiria\Security\ClaimType;
use Aphiria\Security\IPrincipal;

final class MinimumAgeRequirementHandler implements IAuthorizationRequirementHandler
{
    public function function handle(
        IPrincipal $user,
        object $requirement,
        AuthorizationContext $authorizationContext
    ): void {
        if (!$requirement instanceof MinimumAgeRequirement) {
            throw new \InvalidArgumentException('Requirement must be of type ' . MinimumAgeRequirement::class);
        }
        
        $dateOfBirthClaims = $user->getClaims(ClaimType::DateOfBirth);
        
        if (\count($dateOfBirthClaims) !== 1) {
            $authorizationContext->fail();
            
            return;
        }
        
        if ($dateOfBirthClaims[0]->value->diff(new \DateTime('now'))->y < $requirement->minimumAge) {
            $authorizationContext->fail();
            
            return;
        }
        
        // We have to explicitly mark this requirement as having passed
        $authorizationContext->requirementPassed($requirement);
    }
}
```

Finally, let's register this requirement handler and use it in a policy:

```php
use Aphiria\Authorization\AuthorityBuilder;
use Aphiria\Authorization\AuthorizationPolicy;

$authority = (new AuthorityBuilder())
    ->withRequirementHandler(MinimumAgeRequirement::class, new MinimumAgeRequirementHandler())
    ->withPolicy(new AuthorizationPolicy('age-check', new MinimumAgeRequirement(13)))
    ->build();
```

> **Note:** By default, authorization will continue even if there was a failure.  If you want to change it to stop the moment there's a failure, call `AuthorityBuilder::withContinueOnFailure(false)`.

Now, we can use this policy through an attribute:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Authorization\Attributes\AuthorizePolicy;
use Aphiria\Net\Http\IResponse;
use Aphiria\Routing\Attributes\Post;

final class RentalController extends Controller
{
    #[Post('/rentals')]
    #[AuthorizePolicy('age-check')]
    public function createRental(Rental $rental): IResponse
    {
        // ...
    }
}
```

Similarly, we could authorize this by composing `IAuthority`:

```php
use Aphiria\Authentication\Attributes\Authenticate;
use Aphiria\Authentication\IUserAccessor;
use Aphiria\Authorization\IAuthority;

#[Authenticate]
final class RentalController extends Controller
{
    public function __construct(private IAuthority $authority, private IUserAccessor $userAccessor) {}

    #[Post('/rentals')]
    public function createRental(Rental $rental): IResponse
    {
        $user = $this->userAccessor->getUser($this->request);
    
        if (!$this->authority->authorize($user, 'age-check')->passed) {
            return $this->forbidden();
        }
        
        // ...
    }
}
```

<h2 id="configuring-an-authority">Configuring an Authority</h2>

There are two recommended ways of creating your authority.  If you're using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, the authority will automatically be created for you in a [binder](dependency-injection.md#binders).  All you have to do is configure it in `GlobalModule`:

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Authorization\AuthorizationPolicy;
use Aphiria\Authorization\RequirementHandlers\RolesRequirement;
use Aphiria\Authorization\RequirementHandlers\RolesRequirementHandler;
use Aphiria\Framework\Application\AphiriaModule;

final class GlobalModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        // Register a policy
        $this->withAuthorizationPolicy(
            $appBuilder,
            new AuthorizationPolicy(
                'roles',
                new RolesRequirement('admin'),
                'cookie'
            )
        );
        
        // Register a requirement handler
        $this->withAuthorizationRequirementHandler(
            $appBuilder,
            RolesRequirement::class,
            new RolesRequirementHandler()
        );
    }
}
```

> **Note:** Can you can configure whether you want the authority to continue checking requirements after a failure by setting the `aphiria.authorization.continueOnFailure` [config setting](configuration.md) in the skeleton app's _config.php_.

Then, any time you use the `#[AuthorizePolicy]` or `#[AuthorizeRoles]` attributes or `IAuthority`, you'll be able to use your policies.

If you are not using the skeleton app, use `AuthorityBuilder` to configure and build your authorities:

```php
use Aphiria\Authorization\AuthorityBuilder;
use Aphiria\Authorization\AuthorizationPolicy;
use Aphiria\Authorization\RequirementHandlers\RolesRequirement;
use Aphiria\Authorization\RequirementHandlers\RolesRequirementHandler;

$authority = (new AuthorityBuilder())
    ->withPolicy(new AuthorizationPolicy(
        'roles',
        new RolesRequirement('admin'),
        'cookie'
    ))
    ->withRequirementHandler(RolesRequirement::class, new RolesRequirementHandler())
    // Choose whether we want to continue checking requirements after a failure
    ->withContinueOnFailure(false)
    ->build();
```

<h2 id="resource-authorization">Resource Authorization</h2>

Resource authorization is the process of checking if a user has the authority to take an action on a particular resource.  For example, if we have built a comment section, and want users to be able to delete their own comments and admins to be able to delete anyone's, we can use resource authorization to do so.

First, let's start by defining a requirement for our policy:

```php
final class AuthorizedDeleterRequirement
{
    public function __construct(public readonly array $authorizedRoles) {}
}
```

Next, let's define a handler for this requirement:

```php
use Aphiria\Authorization\AuthorizationContext;
use Aphiria\Authorization\IAuthorizationRequirementHandler;
use Aphiria\Security\ClaimType;
use Aphiria\Security\IPrincipal;

final class AuthorizedDeleterRequirementHandler implements IAuthorizationRequirementHandler
{
    public function function handle(
        IPrincipal $user,
        object $requirement,
        AuthorizationContext $authorizationContext
    ): void {
        if (!$requirement instanceof AuthorizedDeleterRequirement) {
            throw new \InvalidArgumentException('Requirement must be of type ' . AuthorizedDeleterRequirement::class);
        }
    
        $comment = $authorizationContext->resource;
    
        // We'll assume Comment is a class in our application
        if (!$comment instanceof Comment) {
            throw new \InvalidArgumentException('Resource must be of type ' . Comment::class);
        }
        
        if ($comment->authorId === $user->getPrimaryId()?->getNameIdentifier()) {
            // The deleter of the comment is the comment's author
            $context->requirementPassed($requirement);
            
            return;
        }
        
        foreach ($requirement->authorizedRoles as $authorizedRole) {
            if ($user->hasClaim(ClaimType::Role, $authorizedRole)) {
                // This user had one of the authorized roles
                $context->requirementPassed($requirement);
                
                return;
            }
        }
        
        // This requirement failed
        $context->fail();
    }
}
```

Now, let's register this policy:

```php
use Aphiria\Authorization\AuthorityBuilder;
use Aphiria\Authorization\AuthorizationPolicy;

$authority = (new AuthorityBuilder())
    ->withRequirementHandler(AuthorizedDeleterRequirement::class, new AuthorizedDeleterRequirementHandler())
    ->withPolicy(new AuthorizationPolicy('authorized-deleter', new AuthorizedDeleterRequirement(['admin'])))
    ->build();
```

Finally, let's use `IAuthority` to do resource authorization in our controller:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Authentication\Attributes\Authenticate;
use Aphiria\Authentication\IUserAccessor;
use Aphiria\Authorization\IAuthority;
use Aphiria\Net\Http\IResponse;
use Aphiria\Routing\Attributes\Delete;

#[Authenticate]
final class CommentController extends Controller
{
    public function __construct(
        private ICommentRepository $comments,
        private IAuthority $authority,
        private IUserAccessor $userAccessor
    ) {
    }

    #[Delete('/comments/:id')]
    public function deleteComment(int $id): IResponse
    {
        if (($comment = $this->comments->getById($id)) === null) {
            return $this->notFound();
        }
    
        $user = $this->userAccessor->getUser($this->request);
    
        if (!$this->authority->authorize($user, 'authorized-deleter', $comment)->passed) {
            return $this->forbidden();
        }
    
        // ...
    }
}
```

<h2 id="authorization-results">Authorization Results</h2>

Authorization returns an instance of `AuthorizationResult`.  You can grab info about whether or not it was successful:

```php
$authorizationResult = $authority->authorize($user, 'some-policy');

if (!$authorizationResult->passed) {
    print_r($authorizationResult->failedRequirements);
}
```

<h2 id="customizing-failed-authorization-responses">Customizing Failed Authorization Responses</h2>

By default, authorization done in the `Authorize` middleware invokes [`IAuthenticator::challenge()`](authentication.md) when a user is not authenticated, and `IAuthenticator::forbid()` when they are not authorized.  If you would like to customize these responses, simply override `Authorize::handleUnauthenticatedUser()` and `Authorize::handleFailedAuthorizationResult()`.

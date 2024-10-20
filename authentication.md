<h1 id="doc-title">Authentication</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Introduction](#introduction)
   1. [Principals](#principals) 
   2. [Claims](#claims)
   3. [Builder](#builder)
3. [Authentication Schemes](#authentication-schemes)
   1. [Default Scheme](#default-scheme)
   2. [Options](#scheme-options)
   3. [Basic Authentication](#basic-authentication)
   4. [Cookie Authentication](#cookie-authentication)
4. [Configuring an Authenticator](#configuring-an-authenticator)
5. [Authentication Results](#authentication-results)
6. [Customizing Authentication Failure Responses](#customizing-authentication-failure-responses)
7. [User Accessors](#user-accessors)
8. [Mocking Authentication](#mocking-authentication)

</div>

</nav>

<h2 id="introduction">Introduction</h2>

Authentication is the process of verifying an identity.  Aphiria's authentication library allows for full customization to suit your application's needs.  At a high level, authentication uses named [schemes](#authentication-schemes) to authenticate [principals](#principals).  Each scheme gets a handler that actually performs the authentication logic, along with options for things like cookie names and lifetimes, login page paths to redirect to on authentication failure, etc.

You can enforce authentication either through attributes on a controller or a specific controller method:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Authentication\Attributes\Authenticate;

#[Authenticate]
final class UserController extends Controller
{
    // ...
}
```

or by composing `IAuthenticator` in your controllers:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Authentication\IAuthenticator;
use Aphiria\Routing\Attributes\Get;

final class UserController extends Controller
{
    public function __construct(private IAuthenticator $authenticator) {}
    
    #[Get('/users/:id')]
    public function getUserById(int $id): User
    {
        $authenticationResult = $this->authenticator->authenticate($this->request);
        
        if (!$authenticationResult->passed) {
            return $this->unauthorized();
        }
    
        // ...
    }
}
```

The `#[Authenticate]` attribute can also take in a [scheme name](#authentication-schemes) or list of scheme names parameter if you wish to use a specific scheme:

```php
#[Authenticate(schemeNames: ['cookie', 'bearer'])]
final class UserController extends Controller
{
    // ...
}
```

> **Note:** Authentication will only fail if _all_ authentication scheme names fail to authenticate.  When this happens, `IAuthenticator::challenge()` will be called for each scheme.  If authentication passes for any scheme, the principal's identities will be merged with all other successfully authenticated schemes.

Likewise, you can specify a scheme to authenticate against in `IAuthenticator::authenticate()`:

```php
$authenticationResult = $this->authenticator->authenticate($this->request, schemeNames: 'cookie');
```

> **Note:** Not specifying a scheme name will cause authentication to use the [default scheme](#default-scheme).  Also, identical to using the attribute for authentication, if you make multiple calls to `IAuthenticator::authenticate()` but with different scheme names, the principal returned in the result will contain all authenticated identities merged together.

We'll go into more details about how to customize responses when authentication does not pass [below](#customizing-authentication-failure-responses).

<h3 id="principals">Principals</h3>

Before we go too far into authentication, let's first go over some terminology.  A principal is the thing, usually a user or system process, that interacts with your application.  You can authenticate a principal and authorize actions performed by it.  A principal contains one or more identities, each of which may contain [claims](#claims).

> **Example:** Let's say you're going to the airport to board a flight.  You, the principal, will be asked to prove you are who you claim to be, which you'll do with your passport - an identity that contains claims about your citizenship, name, and date of birth.  At security, you'll be asked to re-prove your identity with your passport, and also be asked to prove that you have a ticket - another identity that contains claims about your name, your flight number, and assigned seat.  Airport security will authenticate your passport and ticket, and verify that you're authorized to board the flight.

It should be noted that, in Aphiria, a principal is a generic representation of a user, and is solely meant for authentication and authorization.  Your application will likely have its own abstractions for users, and those abstractions' data can be used to populate identities and claims of a principal.  We'll also use the terms "user" and "principal" interchangeably from here on out.

<h3 id="claims">Claims</h3>

To create a principal, you first create its claims for its identities.  A claim is a statement about an identity, eg email, date of birth, roles, etc.  Let's look at an example:

```php
use Aphiria\Security\{Claim, ClaimType, Identity, User};

// Claims data is usually stored in a database
$claimsIssuer = 'example.com';
// Claim types may either be ClaimType enum values or your own custom strings
$claims = [
    // This claim stores the user's ID
    new Claim(ClaimType::NameIdentifier, 123, $claimsIssuer),
    // This claim stores the user's name
    new Claim(ClaimType::Name, 'Dave', $claimsIssuer),
    // This claim stores the user's roles
    new Claim(ClaimType::Role, 'admin', $claimsIssuer)
];
// You can also pass in an array of identities
$user = new User(new Identity($claims));
```

> **Note:** You can have multiple claims of the same type in one identity.  For example, you might have multiple roles for a user, each of which would have its own distinct claim.

`IPrincipal` contains some useful methods for aggregating claims made by all its identities:

```php
$allClaims = $user->claims;
$allRoleClaims = $user->filterClaims(ClaimType::Role);

// Check that not only do they have a role claim type, but that its value is "admin"
if ($user->hasClaim(ClaimType::Role, 'admin')) {
    // ...
}

// Add another identity
$identity = new Identity([new Claim(ClaimType::Email, 'foo@example.com', 'example.com')]);
$user->addIdentity($identity);
```

You can also loop through all the user's identities and query them directly:

```php
foreach ($user->identities as $identity) {
    $allClaims = $identity->claims;
    $allRoleClaims = $identity->filterClaims(ClaimType::Role);
    
    if ($identity->hasClaim(ClaimType::Role, 'admin')) {
        // ...
    }
    
    // A convenience method for grabbing this identity's username
    $username = $identity->name;
    // Another convenience method for grabbing this identity's ID
    $id = $identity->nameIdentifier;
}
```

You'll frequently find yourself dealing with a user's primary identity when checking claims.  Grabbing it is easy.

```php
$primaryIdentity = $user->primaryIdentity;
```

By default, this is the first identity added to the user, but it can be customized with a callback to determine the primary identity.

```php
// This will make the last added identity the primary one
$primaryIdentitySelector = fn (array $identities): ?IIdentity => $identities[\count($identities) - 1] ?? null;
$user = new User($claims, $primaryIdentitySelector);
$userId = $user->primaryIdentity?->nameIdentifier;
```

Typically, the process of authentication will create the principal and store it, usually as a [request](http-requests.md) property.

<h3 id="builder">Builder</h3>

Aphiria provides a fluent builder syntax for principals and identities.  For example, the code in the [claims documentation](#claims) can be simplified to:

```php
use Aphiria\Security\PrincipalBuilder;

$user = new PrincipalBuilder('example.com')->withNameIdentifier(123)
    ->withName('Dave')
    ->withRoles('admin')
    ->build();
```

This will build a primary identity with the specified claims.  The following fluent methods are available to build your identities in `Principal`:

* `withActor(string $value, ?string $issuer)`
* `withAuthenticationSchemeName(string $authenticationSchemeName)`
  * This must be set for the claims to be considered authenticated
* `withClaims(Claim|Claim[] $claims)`
* `withCountry(string $value, ?string $issuer)`
* `withDateOfBirth(DateTimeInterface $value, ?string $issuer)`
* `withDns(string $value, ?string $issuer)`
* `withEmail(string $value, ?string $issuer)`
* `withGender(string $value, ?string $issuer)`
* `withGivenName(string $value, ?string $issuer)`
* `withHomePhone(string $value, ?string $issuer)`
* `withLocality(string $value, ?string $issuer)`
* `withMobilePhone(string $value, ?string $issuer)`
* `withName(string $value, ?string $issuer)`
* `withNameIdentifier(mixed $value, ?string $issuer)`
* `withOtherPhone(string $value, ?string $issuer)`
* `withPostalCode(string|int $value, ?string $issuer)`
* `withRoles(string|string[] $value, ?string $issuer)`
  * If you specify multiple roles, a unique claim for each role will be created with claim type `ClaimType::Role`
* `withRsa(string $value, ?string $issuer)`
* `withSid(string $value, ?string $issuer)`
* `withStateOrProvince(string $value, ?string $issuer)`
* `withStreetAddress(string $value, ?string $issuer)`
* `withSurname(string $value, ?string $issuer)`
* `withThumbprint(string $value, ?string $issuer)`
* `withUpn(string $value, ?string $issuer)`
* `withUri(string $value, ?string $issuer)`
* `withX500DistinguishedName(string $value, ?string $issuer)`

> **Note:** If you pass in an issuer into any of the above methods, it will supersede the default claims issuer set in the `PrincipalBuilder` constructor.  An issuer must be set either in `PrincipalBuilder::__construct()` or in the claim builder methods above.

If you want to build multiple identities for your principal, you can.

```php
use Aphiria\Security\IdentityBuilder;
use Aphiria\Security\PrincipalBuilder;

$user = new PrincipalBuilder('example.com')
    ->withIdentity(function (IdentityBuilder $identity) {
        $identity->withName('Dave');
    })
    ->withIdentity(function (IdentityBuilder $identity) {
        $identity->withThumbprint('abc123');
    })
    ->build();
```

`IdentityBuilder` provides the same fluent methods as `PrincipalBuilder` above.

You can also specify a primary identity selector via `PrincipalBuilder::withPrimaryIdentitySelector()` and passing in a `Closure` like the example [above](#claims).

<h2 id="authentication-schemes">Authentication Schemes</h2>

Now that we've clarified some terminology, let's dive into authentication schemes.  An authentication scheme defines a particular way of authenticating an identity, eg basic authentication.  Each scheme has a name, the name of its handler class, and options.  Handlers implement `IAuthenticationSchemeHandler`, which provides several methods:

* `authenticate(IRequest $request, AuthenticationScheme $scheme): AuthenticationResult`
    * Attempts to authenticate credentials passed via an HTTP request and create a [principal](#principals)
* `challenge(IRequest $request, IResponse $response, AuthenitcationScheme $scheme): void`
    * In the case that authentication fails, this is called to decorate the response to let the user know they could not successfully be authenticated, eg by redirecting to a login page or setting the status code to 401
* `forbid(IRequest $request, IResponse $response, AuthenitcationScheme $scheme): void`
    * In the case that an authenticated user attempted to access a resource they do not have permission to access, this is called to decorate the response to let them know their request was forbidden, eg by redirecting to the forbidden page or setting the status code to 403

Logging in involves passing data along in subsequent requests to help the application authenticate a user.  If you can log in with a particular scheme, its handler should implement `ILoginAuthenticationSchemeHandler`, which defines two additional methods:

* `logIn(IPrincipal $user, IRequest $request, IResponse $response, AuthenticationScheme $scheme): void`
    * Logs a user in and decorates the response with data, eg a cookie, to keep a user logged in for subsequent requests.  `authenticate()` should be called prior to `logIn()` to make sure the credentials were valid.
* `logOut(IRequest $request, IResponse $response, AuthenticationScheme $scheme): void`
    * Logs a user out and decorates the response to remove any data that was used to keep a user logged in

We'll go into more detail on how to register an authentication scheme in the [example below](#basic-authentication).

<h3 id="default-scheme">Default Scheme</h3>

You can register a scheme to be your application's default.  This means that any authentication that does not use a specific scheme will fall back to using the default one.  Registering a default scheme is simple - just pass in `true` as the second parameter to `AuthenticatorBuilder::withScheme()`:

```php
use Aphiria\Authentication\AuthenticationScheme;
use Aphiria\Authentication\AuthenticatorBuilder;

$authenticator = new AuthenticatorBuilder()
    // ...
    ->withScheme(new AuthenticationScheme('cookie', MyCookieHandler::class), true)
    ->build();
```

<h3 id="scheme-options">Options</h3>

All scheme handlers accept a derived class of `AuthenticationSchemeOptions` to help you configure your authentication.  By default, options contain a claims issuer property to help populate your [claims](#claims), but you may extend `AuthenticationSchemeOptions` and add any additional configurable properties you may need, eg cookie names, lifetimes, paths, domains, login page paths, and forbidden page paths.  We'll go into some examples below.

<h3 id="basic-authentication">Basic Authentication</h3>

<a href="https://tools.ietf.org/html/rfc7617" target="_blank">Basic authentication</a> uses the `Authorize` request header with a base64-encoded `username:password` value.  Aphiria provides the base class `BasicAuthenticationHandler` with one abstract method `createAuthenticationResultFromCredentials()` for you to implement.  `BasicAuthenticationHandler` uses `BasicAuthenticationOptions` to configure the <a href="https://tools.ietf.org/html/rfc7235#section-2.2" target="_blank">realm</a> that authentication is valid in.

Let's look at an example concrete implementation of this handler:

```php
use Aphiria\Authentication\{AuthenticationResult, AuthenticationScheme};
use Aphiria\Authentication\Schemes\BasicAuthenticationHandler;
use Aphiria\Net\Http\Request;
use Aphiria\Security\{Claim, ClaimType, Identity, User};
use PDO;

final class SqlBasicAuthenticationHandler extends BasicAuthenticationHandler
{
    public function __construct(private PDO $pdo) {}
    
    protected function createAuthenticationResultFromCredentials(
        string $username,
        string $password,
        IRequest $request,
        AuthenticationScheme $scheme
    ): AuthenticationResult {
        $sql = <<<SQL
SELECT id, email, hashed_password, array_to_json(roles) AS roles FROM users
WHERE LOWER(email) = :email
SQL;
        $statement = $this->pdo->prepare($sql);
        $statement->execute(['email' => \strtolower(\trim($username))]);
        $row = $statement->fetch(PDO::FETCH_ASSOC);
        
        if (
            $statement->rowCount() !== 1
            || !\password_verify($password, $row['hashed_password'])
        ) {
            return AuthenticationResult::fail('Invalid credentials', $scheme->name);
        }
        
        $claimsIssuer = $scheme->options->claimsIssuer ?? $scheme->name;
        $claims = [
            new Claim(ClaimType::NameIdentifier, $row['id'], $claimsIssuer),
            new Claim(ClaimType::Name, $row['email'], $claimsIssuer)
        ];
        
        foreach (\json_decode($row['roles']) as $role) {
            $claims[] = new Claim(ClaimType::Role, $role, $claimsIssuer);
        }
        
        $user = new User(new Identity($claims, $scheme->name));
        
        return AuthenticationResult::pass($user, $scheme->name);
    }
}
```

Let's register this scheme with the authenticator:

```php
use Aphiria\Authentication\AuthenticationScheme;
use Aphiria\Authentication\AuthenticatorBuilder;
use Aphiria\Authentication\Schemes\BasicAuthenticationOptions;

$authenticator = new AuthenticatorBuilder()
    // ...
    ->withScheme(new AuthenticationScheme(
        'basic',
         SqlBasicAuthenticationHandler::class,
         new BasicAuthenticationOptions(realm: 'example.com', claimsIssuer: 'https://example.com')
     ))
    ->build();
```

<h3 id="cookie-authentication">Cookie Authentication</h3>

In the case that you are using cookie values to authenticate, you can extend `CookieAuthenticationHandler` and define the methods `createAuthenticationResultFromCookie()` and `createCookieValueForUser()` to create an authentication result from a cookie value and to create the cookie value that will be used to authenticate in subsequent requests, respectively.  This handler uses `CookieAuthenticationOptions` to give you control over your cookies.

We won't go over how extend `CookieAuthenticationHandler` because it is very similar to the [example above](#basic-authentication), but here is how we would register our implementation:

```php
use Aphiria\Authentication\AuthenticationScheme;
use Aphiria\Authentication\AuthenticatorBuilder;
use Aphiria\Authentication\Schemes\CookieAuthenticationOptions;
use Aphiria\Net\Http\Headers\SameSiteMode;

$authenticator = new AuthenticatorBuilder()
    // ...
    ->withScheme(new AuthenticationScheme(
        'cookie',
         MyCookieAuthenticationHandler::class,
         new CookieAuthenticationOptions(
            cookieName: 'authToken',
            cookieMaxAge: 360,
            cookiePath: '/',
            cookieDomain: 'example.com',
            cookieIsSecure: true,
            cookieIsHttpOnly: true,
            cookieSameSite: SameSiteMode::Strict,
            loginPagePath: '/login',
            forbiddenPagePath: '/access-denied',
            claimsIssuer: 'https://example.com'
         )
     ))
    ->build();
```

Now, whenever we use our `cookie` scheme, cookies will be set using the above options, and redirects on challenges and forbidden requests will forward to the appropriate paths.

<h2 id="configuring-an-authenticator">Configuring an Authenticator</h2>

There are two recommended ways of creating your authenticator.  If you're using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, the authenticator will automatically be created for you in a [binder](dependency-injection.md#binders).  All you have to do is configure it in `GlobalModule`:

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Authentication\AuthenticationScheme;
use Aphiria\Authentication\AuthenticationSchemeOptions;
use Aphiria\Framework\Application\AphiriaModule;

final class GlobalModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        $this->withAuthenticationScheme($appBuilder, new AuthenticationScheme(
            'token',
            MyTokenHandler::class,
            new AuthenticationSchemeOptions(claimsIssuer: 'https://example.com')
        ));
    }
}
```

Then, any time you use the `#[Authenticate]` attribute or `IAuthenticator`, you'll be able to use your `token` scheme.

If you are not using the skeleton app, the simplest method is to use `AuthenticatorBuilder` to configure and build your authenticator:

```php
use Aphiria\Authentication\AuthenticationScheme;
use Aphiria\Authentication\AuthenticationSchemeOptions;
use Aphiria\Authentication\AuthenticatorBuilder;
use Aphiria\Authentication\ContainerAuthenticationSchemeHandlerResolver;
use Aphiria\DependencyInjection\Container;

$authenticator = new AuthenticatorBuilder()
    // This will resolve our scheme handler instances
    ->withHandlerResolver(new ContainerAuthenticationSchemeHandlerResolver(new Container()))
    ->withScheme(
        'token', 
        MyTokenHandler::class,
        new AuthenticationSchemeOptions(claimsIssuer: 'https://example.com')
    )
    ->build();
```

<h2 id="authentication-results">Authentication Results</h2>

An `AuthenticationResult` is returned when authenticating.  To indicate successful authentication, simply call

```php
use Aphiria\Authentication\AuthenticationResult;

// $user is the authenticated principal
$result = AuthenticationResult::pass($user, $scheme->name);
```

Likewise, you can indicate a failure by calling

```php
$result = AuthenticationResult::fail('Invalid credentials', $scheme->name);

// Or pass in an exception instead
$result = AuthenticationResult::fail(new InvalidCredentialException(), $scheme->name);
```

You can grab info about the result:

```php
if (!$result->passed) {
    throw $result->failure;
}

$user = $result->user;
```

<h2 id="customizing-authentication-failure-responses">Customizing Authentication Failure Responses</h2>

By default, when authentication in the `Authenticate` middleware fails, `challenge()` will be called on the same scheme handler that we attempted to authenticate with.  Most handlers' `challenge()` methods will set the status code to 401 or redirect you to the login page, depending on the implementation.  If you'd like to customize this, you can extend `Authenticate` and override `handleFailedAuthenticationResult()` to return a response.

<h2 id="user-accessors">User Accessors</h2>

Once you have authenticated a principal using the `Authenticate` middleware, you can store and retrieve that principal for the duration of the request using `IUserAccessor`.  By default, `RequestPropertyUserAccessor` will be used to store the principal as a [custom property](http-requests.md#basics) on the request.  If you need to access the principal in your controller, simply call `$this->user`:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Authentication\Attributes\Authenticate;
use Aphiria\Routing\Attributes\Delete;

#[Authenticate]
final class BookController extends Controller
{
    #[Delete('/books/:id')]
    public function deleteBook(int $id): void
    {
        $user = $this->user;
        
        // ...
    }
}
```

<h2 id="mocking-authentication">Mocking Authentication</h2>

You can learn how to mock authentication in your tests [here](testing-apis.md#mocking-authentication).

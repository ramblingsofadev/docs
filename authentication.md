<h1 id="doc-title">Authentication</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Introduction](#introduction)
2. [Authentication Schemes](#authentication-schemes)
    1. [Default Scheme](#default-scheme)
    2. [Options](#scheme-options)
    3. [Basic Authentication](#basic-authentication)
    4. [Cookie Authentication](#cookie-authentication)
3. [Authentication Results](#authentication-results)
4. [Customizing Authentication Failure Responses](#customizing-authentication-failure-responses)
5. [User Accessors](#user-accessors)

</div>

</nav>

<h2 id="introduction">Introduction</h2>

Authentication is the process of verifying an identity.  Aphiria's authentication library builds on top of the concepts in its [security library](security.md), and allows for full customization to suit your application's needs.  At a high level, authentication uses named [schemes](#authentication-schemes) to authenticate users.  Each scheme gets a handler that actually performs the authentication logic, along with options for things like cookie names and lifetimes, login page paths to redirect to on authentication failure, etc.

You can enforce authentication either through attributes on a controller or a specific controller method:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Authentication\Attributes\Authenticate;

#[Authenticate]
class UserController extends Controller
{
    // ...
}
```

or by composing `IAuthenticator` in your controllers:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Authentication\IAuthenticator;
use Aphiria\Routing\Attributes\Get;

class UserController extends Controller
{
    public function __construct(private IAuthenticator $authenticator) {}
    
    #[Get('/users/:id')]
    public function getUser(int $id): User
    {
        $authenticationResult = $this->authenticator->authenticate($this->request);
        
        if (!$authenticationResult->passed) {
            return $this->unauthorized();
        }
    
        // ...
    }
}
```

The `#[Authenticate]` attribute can also take in a [scheme name](#authentication-schemes) parameter if you wish to use a specific scheme:

```php
#[Authenticate(schemeName: 'cookie')]
class UserController extends Controller
{
    // ...
}
```

Likewise, you can specify a scheme to authenticate against in `IAuthenticator::authenticate()`:

```php
$authenticationResult = $this->authenticator->authenticate($this->request, schemeName: 'cookie');
```

> **Note:** Not specifying a scheme name will cause authentication to use the [default scheme](#default-scheme).

We'll go into more details about how to customize responses when authentication does not pass [below](#customizing-authentication-failure-responses).

<h2 id="authentication-schemes">Authentication Schemes</h2>

An authentication scheme defines a particular way of authenticating an identity, eg basic authentication.  Each scheme has a name, the name of its handler class, and options.  Handlers implement `IAuthenticationSchemeHandler`, which provides several methods:

* `authenticate(IRequest $request, AuthenticationScheme $scheme): AuthenticationResult`
    * Attempts to authenticate credentials passed via an HTTP request and create a [principal](security.md#working-with-principals)
* `challenge(IRequest $request, IResponse $response, AuthenitcationScheme $scheme): void`
    * In the case that authentication fails, this is called to decorate the response to let the user know they could not successfully be authenticated
* `forbid(IRequest $request, IResponse $response, AuthenitcationScheme $scheme): void`
    * In the case that an authenticated user attempted to access a resource they do not have permission to access, this is called to decorate the response to let them know their request was forbidden

If you can log in with a particular scheme, its handler should implement `ILoginAuthenticationSchemeHandler`, which defines two additional methods:

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

$authenticator = (new AuthenticatorBuilder())
    // ...
    ->withScheme(new AuthenticationScheme('cookie', MyCookieHandler::class), true)
    ->build();
```

<h3 id="scheme-options">Options</h3>

All scheme handlers accept a derived class of `AuthenticationSchemeOptions` to help you configure your authentication.  By default, options contain a claims issuer property to help populate your [claims](security.md#working-with-principals), but you may extend `AuthenticationSchemeOptions` and add any additional configurable properties you may need, eg cookie names, lifetimes, paths, domains, login page paths, and forbidden page paths.  We'll go into some examples below.

<h3 id="basic-authentication">Basic Authentication</h3>

<a href="https://tools.ietf.org/html/rfc7617" target="_blank">Basic authentication</a> uses the `Authorize` request header with a base64-encoded `username:password` value.  Aphiria provides the base class `BasicAuthenticationHandler` with one abstract method `createAuthenticationResultFromCredentials()` for you to implement.  `BasicAuthenticationHandler` uses `BasicAuthenticationOptions` to configure the <a href="https://tools.ietf.org/html/rfc7235#section-2.2" target="_blank">realm</a> that authentication is valid in.

Let's look at an example concrete implementation of this handler:

```php
use Aphiria\Authentication\{AuthenticationResult, AuthenticationScheme};
use Aphiria\Authentication\Schemes\BasicAuthenticationHandler;
use Aphiria\Security\{Claim, ClaimType, Identity, User};
use PDO;

final class SqlBasicAuthenticationHandler extends BasicAuthenticationHandler
{
    public function __construct(private PDO $pdo) {}
    
    protected function createAuthenticationResultFromCredentials(string $username,string $password,AuthenticationScheme $scheme): AuthenticationResult
    {
        $statement = $this->pdo->prepare('SELECT id, email, hashed_password, array_to_json(roles) AS roles FROM users WHERE LOWER(email) = 
        :email');
        $statement->execute(['email' => \strtolower(\trim($username))]);
        $row = $statement->fetch(PDO::FETCH_ASSOC);
        
        if ($row === false || \count($row) === 0 || !\password_verify($password, $row['hashed_password'])) {
            return AuthenticationResult::fail('Invalid credentials');
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
        
        return AuthenticationResult::pass($user);
    }
}
```

Let's register this scheme with the authenticator:

```php
use Aphiria\Authentication\AuthenticationScheme;
use Aphiria\Authentication\AuthenticatorBuilder;
use Aphiria\Authentication\Schemes\BasicAuthenticationOptions;

$authenticator = (new AuthenticatorBuilder())
    // ...
    ->withScheme(new AuthenticationScheme(
        'basic',
         SqlBasicAuthenticationHandler::class,
         new BasicAuthenticationOptions(realm: 'example.com', claimsIssuer: 'https://example.com')
     ))
    ->build();
```

<h3 id="cookie-authentication">Cookie Authentication</h3>

In the case that you are using cookie values to authenticate, you can extend `CookieAuthenticationHandler` and define the methods `createAuthenticationResultFromCookie()` and `getCookieValueFromUser()` to create an authentication result from a cookie value and to create the cookie value that will be used to authenticate in subsequent requests, respectively.  This handler uses `CookieAuthenticationOptions` to give you control over your cookies.

We won't go over how extend `CookieAuthenticationHandler` because it is very similar to the [example above](#basic-authentication), but here is how we would register our implementation:

```php
use Aphiria\Authentication\AuthenticationScheme;
use Aphiria\Authentication\AuthenticatorBuilder;
use Aphiria\Authentication\Schemes\CookieAuthenticationOptions;
use Aphiria\Net\Http\Headers\SameSiteMode;

$authenticator = (new AuthenticatorBuilder())
    // ...
    ->withScheme(new AuthenticationScheme(
        'basic',
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

Now, whenever we use our cookie scheme, cookies will be set using the above options, and redirects on challenges and forbidden requests will forward to the appropriate paths.

<h2 id="authentication-results">Authentication Results</h2>

An `AuthenticationResult` is returned when authenticating.  To indicate successful authentication, simply call

```php
use Aphiria\Authentication\AuthenticationResult;

// $user is the authenticated principal
$result = AuthenticationResult::pass($user);
```

Likewise, you can indicate a failure by calling

```php
$result = AuthenticationResult::fail('Invalid credentials');

// Or pass in an exception instead
$result = AuthenticationResult::fail(new InvalidCredentialException());
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

Once you have authenticated a principal using the `Authenticate` middleware, you can store and retrieve that principal for the duration of the request using `IUserAccessor`.  By default, `RequestPropertyUserAccessor` will be used to store the principal as a [custom property](http-requests.md#basics) on the request.  If you need to access the principal in your controller, simply compose `IUserAccessor`:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Authentication\Attributes\Authenticate;
use Aphiria\Authentication\IUserAccessor;
use Aphiria\Routing\Attributes\Delete;

#[Authenticate]
class BookController extends Controller
{
    public function __construct(private IUserAccessor $userAccessor) {}
    
    #[Delete('/books/:id')]
    public function deleteBook(int $id): void
    {
        $user = $this->userAccessor->getUser($this->request);
        
        // ...
    }
}
```

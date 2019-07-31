# HTTP Controllers

## Table of Contents
1. [Basics](#basics)
2. [Parameter Resolution](#parameter-resolution)
   1. [Request Bodies](#request-body-parameters)
   2. [URI Parameters](#uri-parameters)
   3. [Arrays in Request Bodies](#arrays-in-request-bodies)
3. [Parsing Request Data](#parsing-request-data)
4. [Formatting Response Data](#formatting-response-data)
5. [Closure Controllers](#closure-controllers)
6. [Controller Dependencies](#controller-dependencies)
7. [Middleware](#middleware)
   1. [Configuring Middleware](#configuring-middleware)
8. [API Kernel](#api-kernel)
9. [Exception Handling](#exception-handling)
   1. [Customizing Exception Responses](#customizing-exception-responses)
   2. [Logging](#logging)

<h1 id="basics">Basics</h1>

A controller contains the methods that are invoked when a [request comes through](routing.md).  Your controllers can either extend `Controller` or be a [`Closure`](#closure-controllers).  Let's say you needed an endpoint to create a user.  Simple:

```php
use Aphiria\Api\Controllers\Controller;
use App\Users\User;

final class UserController extends Controller
{
    // ...
    
    public function createUser(User $user): User
    {
        $this->userService->createUser($user);

        return $user;
    }
}
```

Aphiria will see the `User` method parameter and [automatically deserialize the request body to an instance of `User`](#parameter-resolution) (which can be a POPO) using [content negotiation](content-negotiation.md).  It will also detect that a `User` object was returned by the method, and create a 200 response whose body is the serialized user object.  It uses content negotiation to determine the media type to (de)serialize to (eg JSON).

You can also be a bit more explicit and return a response yourself.  For example, the following controller method is functionally identical to the previous example:

```php
final class UserController extends Controller
{
    // ...
    
    public function createUser(User $user): IHttpResponseMessage
    {
        $this->userService->createUser($user);

        return $this->ok($user);
    }
}
```

The `ok()` helper method uses a `NegotiatedResponseFactory` to build a response using the current request and [content negotiation](content-negotiation).  You can pass in a POPO as the response body, and the factory will use content negotiation to determine how to serialize it.

The following helper methods come bundled with `Controller`:

* `badRequest()`
* `conflict()`
* `created()`
* `forbidden()`
* `found()`
* `internalServerError()`
* `movedPermanently()`
* `noContent()`
* `notFound()`
* `ok()`
* `unauthorized()`

If your controller method has a `void` return type, a 204 "No Content" response will be created automatically.

If you need access to the current request, use `$this->request` within your controller method.

Setting headers is simple, too:

```php
use Aphiria\Net\Http\HttpHeaders;

final class UserController extends Controller
{
    // ...
    
    public function getUserById(int $userId): IHttpResponseMessage
    {
        $user = $this->userService->getUserById($userId);
        $headers = new HttpHeaders();
        $headers->add('Cache-Control', 'no-cache');
        
        return $this->ok($user, $headers);
    }
}
```

<h1 id="parameter-resolution">Parameter Resolution</h1>

Your controller methods will frequently need to do things like deserialize the request body or read route/query string values.  Aphiria simplifies this process enormously by allowing your method signatures to be expressive.  

<h2 id="request-body-parameters">Request Bodies</h2>

Object type hints are always assumed to be the request body, and can be automatically deserialized to any POPO:

```php
final class UserController extends Controller
{
    // ...
    
    public function createUser(User $user): IHttpResponseMessage
    {
        $this->userService->createUser($user);
        
        return $this->created();
    }
}
```

This works for any media type (eg JSON) that you've registered to your [content negotiator](content-negotiation).

<h2 id="uri-parameters">URI Parameters</h2>

Aphiria also supports resolving scalar parameters in your controller methods.  It will scan route variables, and then, if no matches are found, the query string for scalar parameters.  For example, this method will grab `includeDeletedUsers` from the query string and cast it to a `bool`:

```php
final class UserController extends Controller
{
    // ...
    
    // Assume path and query string is "users?includeDeletedUsers=1"
    public function getAllUsers(bool $includeDeletedUsers): array
    {
        return $this->userService->getAllUsers($includeDeletedUsers);
    }
}
```

Nullable parameters and parameters with default values are also supported.  If a query string parameter is optional, it _must_ be either nullable or have a default value.

<h2 id="arrays-in-request-bodies">Arrays in Request Bodies</h2>

Request bodies might contain an array of values.  Because PHP doesn't support generics or typed arrays, you cannot use type-hints alone to deserialize arrays of values.  However, it's still easy to do within your controller methods:

```php
final class UserController extends Controller
{
    // ...

    public function createManyUsers(): IHttpResponseMessage
    {
        $users = $this->readRequestBodyAs(User::class . '[]');
        $this->userService->createManyUsers($users);
        
        return $this->created();
    }
}
```

<h1 id="parsing-request-data">Parsing Request Data</h1>

Your controllers might need to do more advanced reading of request data, such as reading cookies, reading multipart bodies, or determining the content type of the request.  To simplify this kind of work, an instance of `RequestParser` is set in your controller:

```php
final class JsonPrettifierController extends Controller
{
    // ...

    public function prettifyJson(): IHttpResponseMessage
    {
        if (!$this->requestParser->isJson($this->request)) {
            return $this->badRequest();
        }
        
        $prettyJson = json_encode($this->request->getBody()->readAsString(), JSON_PRETTY_PRINT);
        $response = new Response(200, null, new StringBody($prettyJson));
        
        return $response;
    }
}
```

<h1 id="formatting-response-data">Formatting Response Data</h1>

If you need to write data back to the response, eg cookies or creating a redirect, an instance of `ResponseFormatter` is available in the controller:

```php
final class LoginController extends Controller
{
    // ...

    public function logIn(LoginRequest $loginRequest): IHttpResponseMessage
    {
        $authResults = null;
        
        // Assume this logic resides in your application
        if (!$this->authenticator->tryLogin($loginRequest->username, $loginRequest->password, $authResults)) {
            return $this->unauthorized();
        }
        
        // Write a cookie containing the auth token back to the response
        $response = new Response(200);
        $authTokenCookie = new Cookie('authtoken', $authResults->getAuthToken(), time() + 3600);
        $this->responseFormatter->setCookie($response, $authTokenCookie);
        
        return $response;
    }
}
```

<h1 id="closure-controllers">Closure Controllers</h1>

Sometimes, a controller class is overkill for a route that does very little.  In this case, you can use a `Closure` when defining your routes:

```php
 $routes->map('GET', 'ping')
    ->toClosure(fn () => $this->ok());
```

Closures support the same [parameter resolution](#parameter-resolution) features as controller methods.  Here's the cool part - Aphiria will bind an instance of `Controller` to your closure, which means you can use [all the methods](#controllers), [request parsers](#parsing-request-data), and [response formatters](#formatting-response-data) available inside of `Controller` via `$this`.

<h1 id="controller-dependencies">Controller Dependencies</h1>

The API library provides support for auto-wiring your controllers.  In other words, it can scan your controllers' constructors for dependencies, resolve them, and then instantiate your controllers with those dependencies.  Dependency resolvers simply need to implement `IDependencyResolver`.  To make it easy for users of Opulence's DI container, you can use `ContainerDependencyResolver`.

```php
use Aphiria\Api\ContainerDependencyResolver;
use Opulence\Ioc\Container;

$container = new Container();
$dependencyResolver = new ContainerDependencyResolver($container);
```

Once you've instantiated your dependency resolver, pass it into your [request handler](#request-handlers) for auto-wiring.
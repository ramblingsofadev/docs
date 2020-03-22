<h1 id="doc-title">Controllers</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
2. [Parameter Resolution](#parameter-resolution)
   1. [Request Bodies](#request-body-parameters)
   2. [URI Parameters](#uri-parameters)
   3. [Arrays in Request Bodies](#arrays-in-request-bodies)
   4. [Validating Request Bodies](#validating-request-bodies)
3. [Parsing Request Data](#parsing-request-data)
4. [Formatting Response Data](#formatting-response-data)
5. [Closure Controllers](#closure-controllers)
6. [Controller Dependencies](#controller-dependencies)

</div>

</nav>

<h2 id="basics">Basics</h2>

A controller contains the methods that are invoked when a [request comes through](routing.md).  Your controllers can either extend `Controller` or be a [`Closure`](#closure-controllers).  Let's say you needed an endpoint to get a user.  Simple:

```php
use Aphiria\Api\Controllers\Controller;
use App\Users\User;

final class UserController extends Controller
{
    // ...
    
    public function getUser(int $id): User
    {
        return $this->users->getUserById($id);
    }
}
```

Aphiria will grab the ID from the URI (preference is given to route variables, and then to query string variables).  It will also detect that a `User` object was returned by the method, and create a 200 response whose body is the serialized user object.  It uses [content negotiation](content-negotiation.md) to determine the media type to serialize to (eg JSON).

You can also be a bit more explicit and return a response yourself.  For example, the following controller method is functionally identical to the previous example:

```php
final class UserController extends Controller
{
    // ...
    
    public function getUser(int $id): IHttpResponseMessage
    {
        $user = $this->users->getUserById($id);

        return $this->ok($user);
    }
}
```

The `ok()` helper method uses a `NegotiatedResponseFactory` to build a response using the current request and [content negotiation](content-negotiation.md).  You can pass in a POPO as the response body, and the factory will use content negotiation to determine how to serialize it.

Similarly, Aphiria can [automatically deserialize request bodies](#request-body-parameters).

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
    
    public function getUser(int $id): IHttpResponseMessage
    {
        $user = $this->users->getUserById($id);
        $headers = new HttpHeaders();
        $headers->add('Cache-Control', 'no-cache');
        
        return $this->ok($user, $headers);
    }
}
```

<h2 id="parameter-resolution">Parameter Resolution</h2>

Your controller methods will frequently need to do things like deserialize the request body or read route/query string values.  Aphiria simplifies this process enormously by allowing your method signatures to be expressive.  

<h3 id="request-body-parameters">Request Bodies</h3>

Object type hints are always assumed to be the request body, and can be automatically deserialized to any POPO:

```php
final class UserController extends Controller
{
    // ...
    
    public function createUser(UserDto $userDto): IHttpResponseMessage
    {
        $user = $this->users->createUser($userDto->email, $userDto->password);
        
        return $this->created("users/{$user->id}", $user);
    }
}
```

This works for any media type (eg JSON) that you've registered to your [content negotiator](content-negotiation.md).

<h3 id="uri-parameters">URI Parameters</h3>

Aphiria also supports resolving scalar parameters in your controller methods.  It will scan route variables, and then, if no matches are found, the query string for scalar parameters.  For example, this method will grab `includeDeletedUsers` from the query string and cast it to a `bool`:

```php
final class UserController extends Controller
{
    // ...
    
    // Assume path and query string is "users?includeDeletedUsers=1"
    public function getAllUsers(bool $includeDeletedUsers): array
    {
        return $this->users->getAllUsers($includeDeletedUsers);
    }
}
```

Nullable parameters and parameters with default values are also supported.  If a query string parameter is optional, it _must_ be either nullable or have a default value.

<h3 id="arrays-in-request-bodies">Arrays in Request Bodies</h3>

Request bodies might contain an array of values.  Because PHP doesn't support generics or typed arrays, you cannot use type-hints alone to deserialize arrays of values.  However, it's still easy to do within your controller methods:

```php
final class UserController extends Controller
{
    // ...

    public function createManyUsers(): IHttpResponseMessage
    {
        $users = $this->readRequestBodyAs(User::class . '[]');
        $this->users->createManyUsers($users);
        
        return $this->created();
    }
}
```

<h3 id="validating-request-bodies">Validating Request Bodies</h3>

It's possible to combine the power of the [serialization](serialization.md) and [validation](validation.md) libraries to automatically validate request bodies on every request.  By default, when an invalid request body is detected, a <a href="https://tools.ietf.org/html/rfc7807" target="_blank">problem details</a> response is returned as a 400.  If you'd like to change the response body to something different, you may do so by [changing the exception response factory](global-exception-handler.md#exception-responses) for an `InvalidRequestBodyException`.

If a request body cannot be automatically deserialized, as in the case of [arrays of objects in request bodies](#arrays-in-request-bodies), you must manually perform validation.

```php
use Aphiria\Api\Validation\IRequestBodyValidator;

final class UserController extends Controller
{
    private IRequestBodyValidator $validator;

    public function createManyUsers(): IHttpResponseMessage
    {
        $users = $this->readRequestBodyAs(User::class . '[]');
        $this->validator->validate($this->request, $users);

        // Continue processing the users...
    }
}
```

<h2 id="parsing-request-data">Parsing Request Data</h2>

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
        
        $bodyAsString = $this->request->getBody()->readAsString();
        $prettyJson = json_encode($bodyAsString, JSON_PRETTY_PRINT);
        $headers = new HttpHeaders();
        $headers->add('Content-Type', 'application/json');
        $response = new Response(200, $headers, new StringBody($prettyJson));
        
        return $response;
    }
}
```

<h2 id="formatting-response-data">Formatting Response Data</h2>

If you need to write data back to the response, eg cookies or creating a redirect, an instance of `ResponseFormatter` is available in the controller:

```php
final class LoginController extends Controller
{
    // ...

    public function logIn(LoginDto $login): IHttpResponseMessage
    {
        $authResults = null;
        
        // Assume this logic resides in your application
        if (!$this->authenticator->tryLogin($login->username, $login->password, $authResults)) {
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

<h2 id="closure-controllers">Closure Controllers</h2>

Sometimes, a controller class is overkill for a route that does very little.  In this case, you can use a `Closure` when defining your routes:

```php
 $routeBuilders->get('ping')
    ->toClosure(fn () => $this->ok());
```

Closures support the same [parameter resolution](#parameter-resolution) features as controller methods.  Here's the cool part - Aphiria will bind an instance of `Controller` to your closure, which means you can use [all the methods](#basics), [request parsers](#parsing-request-data), and [response formatters](#formatting-response-data) available inside of `Controller` via `$this`.

<h2 id="controller-dependencies">Controller Dependencies</h2>

The API library provides support for auto-wiring your controllers.  In other words, it can scan your controllers' and middleware's constructors for dependencies and instantiate them.  Dependency resolvers simply need to implement `IDependencyResolver` (`IContainer` already implements it).

```php
use Aphiria\Api\App;
use Aphiria\Api\Router;
use Aphiria\DependencyInjection\Container;

$container = new Container();

// Assume the route matcher was already set up
$router = new Router($routeMatcher, $container);
$app = new App($container, $router);
```

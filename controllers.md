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
5. [Getting the Current User](#getting-the-current-user)

</div>

</nav>

<h2 id="basics">Basics</h2>

A controller contains the methods that are invoked when your app [handles a request](routing.md).  Your controllers must extend `Controller`.  Let's say you needed an endpoint to get a user.  Simple:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Routing\Attributes\Get;
use App\Users\IUserService;
use App\Users\User;

final class UserController extends Controller
{
    public function __construct(private IUserService $users) {}

    #[Get('users/:userId')]
    public function getUserById(int $userId): User
    {
        return $this->users->getById($userId);
    }
}
```

Aphiria will instantiate `UserController` via [dependency injection](dependency-injection.md) and invoke the matched route.  The `$userId` parameter will be set from the URI (preference is given to route variables, and then to query string variables).  It will also detect that a `User` object was returned by the method, and create a 200 response whose body is the serialized user object.  It uses [content negotiation](content-negotiation.md) to determine the media type to serialize to (eg JSON).  The current request is automatically set in the controller, and is accessible via `$this->request`.

You can also be a bit more explicit and return a response yourself.  For example, the following controller method is functionally identical to the previous example:

```php
final class UserController extends Controller
{
    public function __construct(private IUserService $users) {}
    
    #[Get('users/:userId')]
    public function getUserById(int $userId): IResponse
    {
        $user = $this->users->getById($userId);

        return $this->ok($user);
    }
}
```

The `ok()` helper method uses a `NegotiatedResponseFactory` to build a response using the current request and [content negotiation](content-negotiation.md).  You can pass in a POPO as the response body, and the factory will use content negotiation to determine how to serialize it.

Similarly, Aphiria can [automatically deserialize request bodies](#request-body-parameters).

The following helper methods come bundled with `Controller`:

Method | Status Code
------ | ------
`accepted()` | 202
`badRequest()` | 400
`conflict()` | 409
`created()` | 201
`forbidden()` | 403
`found()` | 302
`internalServerError()` | 500
`movedPermanently()` | 301
`noContent()` | 204
`notFound()` | 404
`ok()` | 200
`unauthorized()` | 401

If your controller method has a `void` return type, a 204 "No Content" response will be created automatically.

Setting headers is simple, too:

```php
use Aphiria\Net\Http\Headers;

final class UserController extends Controller
{
    public function __construct(private IUserService $users) {}
    
    #[Get('users/:userId')]
    public function getUserById(int $userId): IResponse
    {
        $user = $this->users->getById($userId);
        $headers = new Headers();
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
    public function __construct(private IUserService $users) {}
    
    #[Post('users')]
    public function createUser(UserDto $userDto): IResponse
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
    
    // Assume the query string is "?includeDeletedUsers=1"
    #[Get('users')]
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

    #[Post('users')]
    public function createManyUsers(): IResponse
    {
        $users = $this->readRequestBodyAs(User::class . '[]');
        $this->users->createManyUsers($users);
        
        return $this->created();
    }
}
```

<h3 id="validating-request-bodies">Validating Request Bodies</h3>

[Aphiria's validation library](validation.md) automatically validates request bodies on every request.  By default, when an invalid request body is detected, a <a href="https://tools.ietf.org/html/rfc7807" target="_blank">problem details</a> response is returned as a 400.  If you'd like to change the response body to something different, you may do so by [creating a custom mapping](exception-handling.md#custom-problem-details-mappings) for an `InvalidRequestBodyException`.

If a request body cannot be automatically deserialized, as in the case of [arrays of objects in request bodies](#arrays-in-request-bodies), you must manually perform validation.

```php
use Aphiria\Api\Validation\IRequestBodyValidator;

final class UserController extends Controller
{
    public function __construct(private IRequestBodyValidator $validator) {}

    #[Post('users')]
    public function createManyUsers(): IResponse
    {
        $users = $this->readRequestBodyAs(User::class . '[]');
        $this->validator->validate($this->request, $users);

        // Continue processing the users...
    }
}
```

<h2 id="parsing-request-data">Parsing Request Data</h2>

Your controllers might need to do more advanced reading of [request data](http-requests.md), such as reading cookies, reading multipart bodies, or determining the content type of the request.  To simplify this kind of work, an instance of `RequestParser` is automatically set in your controller:

```php
final class JsonPrettifierController extends Controller
{
    #[Post('prettyjson')]
    public function prettifyJson(): IResponse
    {
        if (!$this->requestParser->isJson($this->request)) {
            return $this->badRequest();
        }
        
        $bodyAsString = $this->request->body->readAsString();
        $prettyJson = json_encode($bodyAsString, JSON_PRETTY_PRINT);
        $headers = new Headers();
        $headers->add('Content-Type', 'application/json');
        $response = new Response(HttpStatusCode::Ok, $headers, new StringBody($prettyJson));
        
        return $response;
    }
}
```

<h2 id="formatting-response-data">Formatting Response Data</h2>

If you need to write data back to the [response](http-responses.md), eg cookies or creating a redirect, an instance of `ResponseFormatter` is automatically available in the controller:

```php
final class PreferencesController extends Controller
{
    public function __construct(private IPreferenceService $preferences) {}

    #[Put('preferences')]
    public function savePreferences(Preferences $preferences): IResponse
    {
        // Store the preferences
        $this->preferences->save($preferences);
    
        // Write a cookie containing the preferences for a better UX
        $response = new Response();
        $preferencesCookie = new Cookie('preferences', $preferences->toJson(), 60 * 60 * 24 * 30);
        $this->responseFormatter->setCookie($response, $preferencesCookie);
        
        return $response;
    }
}
```

<h2 id="getting-the-current-user">Getting the Current User</h2>

If you're using the [authentication library](authentication.md), you can grab the current [user](authentication.md#principals):

```php
final class BookController extends Controller
{
    #[Get('books/:id')]
    #[Authenticate]
    public function getBook(int $id): Book
    {
        // This can be null if the user was not set by authentication middleware
        $user = $this->user;
        
        // ...
    }
}
```

<h1 id="doc-title">Middleware</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
   1. [Manipulating the Request](#manipulating-the-request)
   2. [Manipulating the Response](#manipulating-the-response)
   3. [Middleware Parameters](#middleware-parameters)
2. [Executing Middleware](#executing-middleware)

</div>

</nav>

<h2 id="basics">Basics</h2>

The middleware library provides developers a way of defining route middleware for their applications.  Middleware are simply layers of request and response processing before and after a controller action is invoked.  This is extremely useful for actions like authorization, logging, and request/response decoration.

Middleware have a simple signature:

```php
use Aphiria\Net\Http\{IRequest, IRequestHandler, IResponse};

interface IMiddleware
{
    public function handle(IRequest $request, IRequestHandler $next): IResponse;
}
```

`IMiddleware::handle()` takes in the current request and

1. Optionally manipulates it
2. Passes the request on to the next request handler in the pipeline
3. Optionally manipulates the response returned by the next request handler
4. Returns the response

<h3 id="manipulating-the-request">Manipulating the Request</h3>

To manipulate the request before it gets to the controller, make changes to it before calling `$next->handle($request)`:

```php
use Aphiria\Middleware\IMiddleware;
use Aphiria\Net\Http\{IRequest, IRequestHandler, IResponse};

final class RequestManipulator implements IMiddleware
{
    public function handle(IRequest $request, IRequestHandler $next): IResponse
    {
        // Do our work before returning $next->handle($request)
        $request->properties->add('Foo', 'bar');

        return $next->handle($request);
    }
}
```

<h3 id="manipulating-the-response">Manipulating the Response</h3>

To manipulate the response after the controller has done its work, do the following:

```php
use Aphiria\Middleware\IMiddleware;
use Aphiria\Net\Http\{IRequest, IRequestHandler, IResponse};

final class ResponseManipulator implements IMiddleware
{
    public function handle(IRequest $request, IRequestHandler $next): IResponse
    {
        $response = $next->handle($request);

        // Make our changes
        $response->headers->add('Foo', 'bar');

        return $response;
    }
}
```

<h3 id="middleware-parameters">Middleware Parameters</h3>

Occasionally, you'll find yourself wanting to pass primitive values to middleware to indicate something such as a required role to execute an action.  In these cases, your middleware should extend `ParameterizedMiddleware`:

```php
use Aphiria\Middleware\ParameterizedMiddleware;
use Aphiria\Net\Http\{IRequest, IRequestHandler, IResponse};

final class RoleMiddleware extends ParameterizedMiddleware
{
    // Inject any dependencies your middleware needs
    public function __construct(private IAuthService $authService) {}

    public function handle(IRequest $request, IRequestHandler $next): IResponse
    {
        $accessToken = null;

        if (
            !$request->headers->tryGetFirst('Authorization', $accessToken)
            || !$this->authService->accessTokenIsValid($accessToken)
        ) {
            return new Response(401);
        }
    
        // Parameters are available via $this->getParameter()
        if (!$this->authService->accessTokenHasRole($accessToken, $this->getParameter('role'))) {
            return new Response(403);
        }

        return $next->handle($request);
    }
}
```

To actually specify `role`, pass it into your [route configuration](routing.md#route-attributes-middleware):

```php
use Aphiria\Routing\Attributes\{Delete, Middleware};

final class UserController extends Controller
{
    #[
        Delete('users/:userId'), 
        Middleware(RoleMiddleware::class, parameters: ['role' => 'admin'])
    ]
    public function deleteUser(int $userId): void
    {
        // ...
    }
}
```

> **Note:** There are two nearly identical attributes for middleware: `Aphiria\Middleware\Attributes\Middleware` and `Aphiria\Routing\Attributes\Middleware`.  If you're using the Aphiria [router](routing.md), they can be used interchangeably.  The middleware library's attribute is useful if you choose not to use the Aphiria router.

<h2 id="executing-middleware">Executing Middleware</h2>

If you're using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, middleware will be executed automatically for you.  You can also define both [global middleware](configuration.md#component-middleware) and [route middleware](routing.md#middleware).

If you're not using the skeleton app, you'll have to set up a pipeline to execute your middleware for you.  Typically, middleware are wrapped in request handlers (eg `MiddlewareRequestHandler`) and executed in a pipeline.  You can create this pipeline using `MiddlewarePipelineFactory`:

```php
use Aphiria\Middleware\MiddlewarePipelineFactory;

// Assume these are defined by your application
$loggingMiddleware = new LoggingMiddleware();
$authMiddleware = new AuthenticationMiddleware();
$controllerHandler = new ControllerRequestHandler();

$pipeline = new MiddlewarePipelineFactory()->createPipeline(
    [$loggingMiddleware, $authMiddleware],
    $controllerHandler
);
``` 

`$pipeline` will itself be a request handler, which you can then send a request through and receive a response:

```php
$request = new RequestFactory()->createRequestFromSuperglobals($_SERVER);
$response = $pipeline->handle($request);
```

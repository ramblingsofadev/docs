<h1 id="doc-title">Routing</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
   1. [Route Variables](#route-variables)
   2. [Optional Route Parts](#optional-route-parts)
   3. [Route Builders](#route-builders)
   4. [Using Aphiria's Net Library](#using-aphirias-net-library)
   5. [Using Aphiria's Application Builder Library](#using-aphirias-application-builder-library)
2. [Route Actions](#route-actions)
3. [Binding Middleware](#binding-middleware)
   1. [Middleware Attributes](#middleware-attributes)
4. [Grouping Routes](#grouping-routes)
5. [Route Attributes](#route-attributes)
  1. [Example](#route-attribute-example)
  2. [Route Groups](#route-attribute-groups)
  3. [Middleware](#route-attribute-middleware)
  4. [Scanning For Attributes](#scanning-for-attributes)
6. [Route Constraints](#route-constraints)
   1. [Example - Versioned API](#versioned-api-example)
   2. [Getting Headers in PHP](#getting-php-headers)
7. [Route Variable Constraints](#route-variable-constraints)
   1. [Built-In Constraints](#built-in-constraints)
   2. [Making Your Own Custom Constraints](#making-your-own-custom-constraints)
8. [Creating Route URIs](#creating-route-uris)
9. [Caching](#caching)
   1. [Route Caching](#route-caching)
   2. [Trie Caching](#trie-caching)
10. [Matching Algorithm](#matching-algorithm)

</div>

</nav>

<h2 id="basics">Basics</h2>

Routing is the process of mapping HTTP requests to actions.  There are so many routing libraries out there.  Why use this one?  Well, there are a few reasons:

* It isn't coupled to _any_ library/framework
* It supports things that other route matching libraries do not support, like:
  * [Binding framework-agnostic middleware to routes](#binding-middleware)
  * [The ability to add custom matching constraints on route variables](#route-variable-constraints)
  * [The ability to match on header values](#versioned-api-example), which makes things like versioning your routes a cinch
* It is fast
  * <a href="https://github.com/aphiria/aphiria/blob/0.x/src/Router/bin/benchmarks.php" target="_blank">With 400 routes, it takes ~0.0025ms to match any route (~200% faster than FastRoute)</a>
  * The speed is due to the unique [trie-based matching algorithm](#matching-algorithm)
* Its [fluent syntax](#route-builders) keeps you from having to memorize how to set up config arrays
* It supports [attributes](#route-attributes) for defining your routes
* It supports [creating URIs from routes](#creating-route-uris)
* It is built to support the latest PHP 8.0 features

Let's look at a fully-functional example:

```php
use Aphiria\Routing\Builders\RouteCollectionBuilder;
use Aphiria\Routing\Matchers\TrieRouteMatcher;
use Aphiria\Routing\UriTemplates\Compilers\Tries\TrieFactory;
use App\Books\Api\{BookController, Authorization};

// Register the routes
$routes = new RouteCollectionBuilder();
$routes->get('/books/:bookId')
    ->mapsToMethod(BookController::class, 'getBooksById')
    ->withMiddleware(Authorization::class);

// Set up the route matcher
$routeMatcher = new TrieRouteMatcher((new TrieFactory($routes->build()))->createTrie());

// Finally, let's find a matching route
$result = $routeMatcher->matchRoute(
    $_SERVER['REQUEST_METHOD'],
    $_SERVER['HTTP_HOST'],
    $_SERVER['REQUEST_URI']
);
```

Let's say the request was `GET /books/123`.  You can check if a match was found by calling:

```php
if ($result->matchFound) {
    // ...
}
```

Grabbing the matched controller info is as simple as:

```php
$result->route->action->controllerName; // "BookController"
$result->route->action->methodName; // "getBooksById"
```

To get the [route variables](#route-variables), call:

```php
$result->routeVariables; // ["bookId" => "123"]
```

To get the [middleware bindings](#binding-middleware), call:

```php
$result->route->middlewareBindings;
```

If `$result->methodIsAllowed` is `false`, you can return a 405 response with a list of allowed methods:

```php
header('Allow', implode(', ', $result->allowedMethods));
```

<h3 id="route-variables">Route Variables</h3>

Aphiria provides a simple syntax for your URIs.  To capture variables in your route, use `:varName`, eg:

```php
users/:userId/profile
```

If you'd like to use [constraints](#route-variable-constraints), then put them in parentheses after the variable:
```php
:varName(constraint1,constraint2(param1,param2))
```

<h3 id="optional-route-parts">Optional Route Parts</h3>

If part of your route is optional, then surround it with brackets.  For example, the following will match both _archives/2017_ and _archives/2017/7_:
```php
archives/:year[/:month]
```

Optional route parts can be nested:

```php
archives/:year[/:month[/:day]]
```

This would match _archives/2017_, _archives/2017/07_, and _archives/2017/07/24_.

<h3 id="route-builders">Route Builders</h3>

Route builders give you a fluent syntax for mapping your routes to controller methods.  They also let you [bind any middleware](#binding-middleware) classes and properties to the route.  The following methods are available to create routes:
  
 ```php
$routes->delete('/foo');
$routes->get('/foo');
$routes->options('/foo');
$routes->patch('/foo');
$routes->post('/foo');
$routes->put('/foo');
```

Each method returns an instance of `RouteBuilder`, and accepts the following parameters:

* `string $pathTemplate`
  * The path for this route ([read about syntax](#route-variables))
* `string|null $hostTemplate` (optional)
  * The optional host template for this route  ([read about syntax](#route-variables))
* `bool $isHttpsOnly` (optional)
  * Whether or not this route is HTTPS-only
  
 You can also call `RouteCollectionBuilder::route()` and pass in the HTTP method(s) you'd like to map to.

<h3 id="using-aphirias-net-library">Using Aphiria's Net Library</h3>

You can use [Aphiria's net library](http-requests.md) to route the request instead of relying on PHP's superglobals:

```php
use Aphiria\Net\Http\RequestFactory;

$request = (new RequestFactory)->createRequestFromSuperglobals($_SERVER);

// Set up your route matcher like before...

$result = $routeMatcher->matchRoute(
    $request->getMethod(),
    $request->getUri()->getHost(),
    $request->getUri()->getPath()
);
```

<h3 id="using-aphirias-application-builder-library">Using Aphiria's Application Builder Library</h3>

Learn more about how [Aphiria's application builder library](configuration.md#component-routes) can simplify registering your routes.

<h2 id="route-actions">Route Actions</h2>

A route action contains the controller method to call when a route is matched.

```php
$routes->get('users/:userId')
    ->mapsToMethod(UserController::class, 'getUserById');
```

<h2 id="binding-middleware">Binding Middleware</h2>

Middleware are a great way to modify both the request and the response on an endpoint.  Aphiria lets you define middleware on your endpoints without binding you to any particular library/framework's middleware implementations.

To bind a single middleware class to your route, call:

```php
$routes->get('foo')
    ->mapsToMethod(MyController::class, 'myMethod')
    ->withMiddleware(FooMiddleware::class);
```

To bind many middleware classes, call:

```php
$routes->get('foo')
    ->mapsToMethod(MyController::class, 'myMethod')
    ->withManyMiddleware([
        FooMiddleware::class,
        BarMiddleware::class
    ]);
```

Under the hood, these class names get converted to instances of `MiddlewareBinding`.  

<h3 id="middleware-attributes">Middleware Attributes</h3>

Some frameworks, such as Aphiria and Laravel, let you bind attributes to middleware.  For example, if you have an `Authorization` middleware, but need to bind the user role that's necessary to access that route, you might want to pass in the required user role.  Here's how you can do it:

```php
$routes->get('foo')
    ->mapsToMethod(MyController::class, 'myMethod')
    ->withMiddleware(Authorization::class, ['role' => 'admin']);

// Or

$routes->get('foo')
    ->mapsToMethod(MyController::class, 'myMethod')
    ->withManyMiddleware([
        new MiddlewareBinding(Authorization::class, ['role' => 'admin']),
        // Other middleware...
    ]);
```

Here's how you can grab the middleware on a matched route:

```php
foreach ($result->middlewareBindings as $middlewareBinding) {
    $middlewareBinding->className; // "Authorization"
    $middlewareBinding->attributes; // ["role" => "admin"]
}
```

<h2 id="grouping-routes">Grouping Routes</h2>

Often times, a lot of your routes will share similar properties, such as hosts and paths to match on, or middleware.  You can group these routes together using `RouteCollectionBuilder::group()` and specifying the options to apply to all routes within the group:

```php
use Aphiria\Routing\Builders\{RouteCollectionBuilder, RouteGroupOptions};

$routes->group(
    new RouteGroupOptions('courses/:courseId', 'example.com'),
    function (RouteCollectionBuilder $routes) {
        // This route's path will use the group's path
        $routes->get('')
            ->mapsToMethod(CourseController::class, 'getCourseById');

        $routes->get('/professors')
            ->mapsToMethod(CourseController::class, 'getCourseProfessors');
    }
);
```

This creates two routes with a host suffix of _example.com_ and a route prefix of _courses/:courseId_ (`example.com/courses/:courseId` and `example.com/courses/:courseId/professors`).  You can set several settings in group options:

```php
use Aphiria\Middleware\MiddlewareBinding;
use Aphiria\Routing\Builders\RouteCollectionBuilder;
use Aphiria\Routing\Builders\RouteGroupOptions;

$routes->group(
    new RouteGroupOptions(
        'courses/:courseId', // The path template
        'api.example.com', // The nullable host template
        true, // Whether or not all routes are HTTPS-only
        [new MyConstraint()], // List of constraints
        ['role' => 'admin'], // The constraint attributes
        [new MiddlewareBinding(Authentication::class)] // The middleware
    ),
    function (RouteCollectionBuilder $routes) {
        // ...
    }
)
```

It is possible to nest route groups.

<h2 id="route-attributes">Route Attributes</h2>

Up until this point, we've shown you how to manually map routes to controllers, but there is another way - attributes.  Aphiria provides the optional functionality to define your routes via attributes if you so choose.  A benefit to defining your routes this way is that it keeps the definition of your routes close (literally) to your controller methods, reducing the need to jump around your code base.

<h3 id="route-attribute-example">Example</h3>

Let's actually define a route:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Net\Http\IResponse;
use Aphiria\Routing\Attributes\{Middleware, Put};
use App\Users\Api\Authorization;
use App\Users\User;

final class UserController extends Controller
{
     #[
        Put('users/:id'),
        Middleware(Authorization::class, attributes: ['role' => 'admin'])
     ]
    public function updateUser(User $user): IResponse
    {
        // ...
    }
}
```

> **Note:** Controllers must either extend `Aphiria\Api\Controllers\Controller` or use the `#[Controller]` attribute.

The following HTTP methods have route attributes: `#[Any]` (any HTTP method), `#[Delete]`, `#[Get]`, `#[Head]`, `#[Options]`, `#[Patch]`, `#[Post]`, `#[Put]`, and `#[Trace]`.  Each attribute takes in the same parameters:

```php
use Aphiria\Routing\Attributes\Get;
use Aphiria\Routing\Attributes\RouteConstraint;

#[
    Get(
        'courses/:courseId',
        host: 'api.example.com',
        name: 'getCourse',
        isHttpsOnly: true,
        attributes: ['role' => 'admin']
    ),
    RouteConstraint(MyConstraint::class, constructorParams: ['param1'])
]
```

<h3 id="route-attribute-groups">Route Groups</h3>

Just like with our [route builders](#grouping-routes), we can also group routes with attributes:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Net\Http\IResponse;
use Aphiria\Routing\Attribute\{Middleware, Put, RouteGroup};
use App\Users\Api\Authorization;
use App\Users\User;

#[RouteGroup(path: 'users')]
class UserController extends Controller
{
     #[
        Put(':id'),
        Middleware(Authorization::class)
     ]
    public function createUser(User $user): IResponse
    {
        // ...
    }
}
```

When our routes get compiled, the route group path will be prefixed to the path of any route within the controller.  In the above example, this would create a route with path `users/:id`.  You can add the following properties to route group attributes:

```php
use Aphiria\Routing\Attributes\RouteConstraint;
use Aphiria\Routing\Attributes\RouteGroup;

#[
    RouteGroup(
        path: 'users',
        host: 'api.example.com',
        isHttpsOnly: true,
        attributes: ['role' => 'admin']
    ),
    RouteConstraint(MyConstraint::class, constructorParams: ['param1'])
]
```
  
<h3 id="route-attribute-middleware">Middleware</h3>

Middleware are added separately:

```php
use Aphiria\Routing\Attributes\Middleware;

#[Middleware(Authorization::class, attributes: ['role' => 'admin'])]
```

<h3 id="scanning-for-attributes">Scanning For Attributes</h3>

Before you can use attributes, you'll need to configure Aphiria to scan for them.  If you're using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, you can do so in `App`:

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Framework\Application\AphiriaComponents;

final class App implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        $this->withRouteAttributes($appBuilder);
    }
}
```

Otherwise, you can manually configure the router to scan for attributes:

```php
use Aphiria\Routing\Attributes\AttributeRouteRegistrant;
use Aphiria\Routing\Matchers\TrieRouteMatcher;
use Aphiria\Routing\RouteCollection;
use Aphiria\Routing\UriTemplates\Compilers\Tries\TrieFactory;

$routes = new RouteCollection();
$routeAttributeRegistrant = new AttributeRouteRegistrant(['PATH_TO_SCAN']);
$routeAttributeRegistrant->registerRoutes($routes);
$routeMatcher = new TrieRouteMatcher((new TrieFactory($routes))->createTrie());

// Find a matching route
$result = $routeMatcher->matchRoute(
    $_SERVER['REQUEST_METHOD'],
    $_SERVER['HTTP_HOST'],
    $_SERVER['REQUEST_URI']
);
```

<h2 id="route-constraints">Route Constraints</h2>

Sometimes, you might find it useful to add some custom logic for matching routes.  This could involve enforcing anything from only allowing certain HTTP methods for a route (eg `HttpMethodRouteConstraint`) or only allowing HTTPS requests to a particular endpoint.  Let's go into some concrete examples...

<h3 id="versioned-api-example">Example - Versioned API</h3>

Let's say your app sends an API version header, and you want to match an endpoint that supports that version.  You could do this by using a route "attribute" and a route constraint.  Let's create some routes that have the same path, but support different versions of the API:

```php
// This route will require an API-VERSION value of 'v1.0'
$routes->get('comments')
    ->mapsToMethod(CommentController::class, 'getAllComments1_0')
    ->withAttribute('API-VERSION', 'v1.0')
    ->withConstraint(new ApiVersionConstraint());

// This route will require an API-VERSION value of 'v2.0'
$routes->get('comments')
    ->mapsToMethod(CommentController::class, 'getAllComments2_0')
    ->withAttribute('API-VERSION', 'v2.0')
    ->withConstraint(new ApiVersionConstraint());
```

> **Note:** If you plan on adding many attributes or constraints to your routes, use `RouteBuilder::withManyAttributes()` and `RouteBuilder::withManyConstraints()`, respectively.

Now, let's add a route constraint to match the "API-VERSION" header to the attribute on our route:

```php
use Aphiria\Routing\Matchers\Constraints\IRouteConstraint;
use Aphiria\Routing\Matchers\MatchedRouteCandidate;

final class ApiVersionConstraint implements IRouteConstraint
{
    public function passes(
        MatchedRouteCandidate $matchedRouteCandidate,
        string $method,
        string $host,
        string $path,
        array $headers
    ): bool {
        $attributes = $matchedRouteCandidate->route->attributes;

        if (!isset($attributes['API-VERSION'])) {
            return false;
        }

        return in_array($attributes['API-VERSION'], $headers['API-VERSION'], true);
    }
}
```

If we hit _/comments_ with an "API-VERSION" header value of "v2.0", we'd match the second route in our example.

<h3 id="getting-php-headers">Getting Headers in PHP</h3>

PHP is irritatingly difficult to extract headers from `$_SERVER`.  If you're using a library/framework to grab headers, then use that.  Otherwise, you can use the `HeaderParser`:

```php
use Aphiria\Routing\Requests\RequestHeaderParser;

$headers = (new RequestHeaderParser)->parseHeaders($_SERVER);
```

<h2 id="route-variable-constraints">Route Variable Constraints</h2>

You can enforce certain constraints to pass before matching on a route.  These constraints come after variables, and must be enclosed in parentheses.  For example, if you want an integer to fall between two values, you can specify a route of

```php
:month(int,min(1),max(12))
```

> **Note:** If a constraint does not require any parameters, then the parentheses after the constraint slug are optional.

<h3 id="built-in-constraints">Built-In Constraints</h3>

The following constraints are built-into Aphiria:

Name | Description
------ | ------
`alpha` | The value must only contain alphabet characters
`alphanumeric` | The value must only contain alphanumeric characters
`between` | The value must fall between a min and max (takes in whether or not the min and max values are inclusive)
`date` | The value must match a date-time format
`in` | The value must be in a list of acceptable values
`int` | The value must be an integer
`notIn` | The value must not be in a list of values
`numeric` | The value must be numeric
`regex` | The value must satisfy a regular expression
`uuidv4` | The value must be a UUID v4

<h3 id="making-your-own-custom-constraints">Making Your Own Custom Constraints</h3>

You can register your own constraint by implementing `IRouteVariableConstraint`.  Let's make a constraint that enforces a certain minimum string length:

```php
use Aphiria\Routing\UriTemplates\Constraints\IRouteVariableConstraint;

final class MinLengthConstraint implements IRouteVariableConstraint
{
    public function __construct(int $minLength) {}

    public static function getSlug(): string
    {
        return 'minLength';
    }

    public function passes($value): bool
    {
        return mb_strlen($value) >= $this->minLength;
    }
}
```

Let's register our constraint with the constraint factory:

```php
use Aphiria\Routing\UriTemplates\Constraints\RouteVariableConstraintFactory;
use Aphiria\Routing\UriTemplates\Constraints\RouteVariableConstraintFactoryRegistrant;

// Register some built-in constraints to our factory
$constraintFactory = (new RouteVariableConstraintFactoryRegistrant)
    ->registerConstraintFactories(new RouteVariableConstraintFactory);

// Register our custom constraint
$constraintFactory->registerConstraintFactory(
    MinLengthConstraint::getSlug(),
    fn (int $minLength) => new MinLengthConstraint($minLength)
);
```

Finally, register this constraint factory with the trie compiler:

```php
use Aphiria\Routing\Builders\RouteCollectionBuilder;
use Aphiria\Routing\Matchers\TrieRouteMatcher;
use Aphiria\Routing\UriTemplates\Compilers\Tries\{TrieCompiler, TrieFactory};

$routes = new RouteCollectionBuilder();
$routes->get('parts/:serialNumber(minLength(6))')
    ->mapsToMethod(PartController::class, 'getPartBySerialNumber');

$trieCompiler = new TrieCompiler($constraintFactory);
$trieFactory = new TrieFactory($routes->build(), null, $trieCompiler);
$routeMatcher = new TrieRouteMatcher($trieFactory->createTrie());
```

Our route will now enforce a serial number with minimum length 6.

<h2 id="creating-route-uris">Creating Route URIs</h2>

You might find yourself wanting to create a link to a particular route within your app.  Let's say you have a route named `GetUserById` with a URI template of `/users/:id`.  We can generate a link to get a particular user:

```php
use Aphiria\Routing\UriTemplates\AstRouteUriFactory;

// Assume you've already created your routes
$routeUriFactory = new AstRouteUriFactory($routes);

// Will create "/users/123"
$uriForUser123 = $routeUriFactory->createRouteUri('GetUserById', ['id' => 123]);
```

Generated URIs will be a relative path unless the URI template specified a host.  Let's look at an example for one that does include a host: `:environment.example.com/users/:id`.

```php
// Will create "https://dev.example.com/users/123"
$uriForDevUser123 = $routeUriFactory->createRouteUri('GetUserById', ['environment' => 'dev', 'id' => 123]);
```

> **Note:**  Absolute URIs are assumed to be HTTPS unless the URI template is specifically set to not be HTTPS-only.

Optional route variables can be specified, too.  Let's assume the URI template is `/archives/:year[/:month]`:

```php
// Will create "/archives/2019"
$booksFor2019 = $routeUriFactory->createRouteUri(
    'GetBooksFromArchive',
    ['year' => 2019]
);

// Will create "/archives/2019/12"
$booksForDec2019 = $routeUriFactory->createRouteUri(
    'GetBooksFromArchive',
    ['year' => 2019, 'month' => 12]
);
```

<h2 id="caching">Caching</h2>

The process of building your routes and compiling the trie is a relatively slow process, and isn't necessary in a production environment where route definitions aren't changing.  Aphiria provides both the ability to cache the results of your route builders and the compiled trie.

<h3 id="route-caching">Route Caching</h3>

To enable caching, pass in an `IRouteCache` (`FileRouteCache` is provided) to the first parameter of `RouteRegistrantCollection`:

```php
use Aphiria\Routing\Caching\FileRouteCache;
use Aphiria\Routing\RouteCollection;
use Aphiria\Routing\RouteRegistrantCollection;

$routes = new RouteCollection();
$routeRegistrant = new RouteRegistrantCollection(new FileRouteCache('/tmp/routeCache.txt'));

// Once you're done configuring your route registrant...

$routeRegistrant->registerRoutes($routes);
```

<h3 id="trie-caching">Trie Caching</h3>

To enable caching, pass in an `ITrieCache` (`FileTrieCache` comes with Aphiria) to your trie factory (passing in `null` will disable caching).  If you want to enable caching for a particular environment, you could do so:

```php
use Aphiria\Routing\UriTemplates\Compilers\Tries\Caching\FileTrieCache;
use Aphiria\Routing\UriTemplates\Compilers\Tries\TrieFactory;

// Let's say that your environment name is stored in an environment var named 'ENV_NAME'
$trieCache = getenv('ENV_NAME') === 'production'
    ? new FileTrieCache('/tmp/trieCache.txt')
    : null;
$trieFactory = new TrieFactory($routes, $trieCache);

// Finish setting up your route matcher...
```

<h2 id="matching-algorithm">Matching Algorithm</h2>

Rather than the typical regex approach to route matching, we decided to go with a <a href="https://en.wikipedia.org/wiki/Trie" target="_blank">trie-based</a> approach.  Each node maps to a segment in the path, and could either contain a literal or a variable value.  We try to proceed down the tree to match what's in the request URI, always giving preference to literal matches over variable ones, even if variable segments are declared first in the routing config.  This logic not only applies to the first segment, but recursively to all subsequent segments.  The benefit to this approach is that it doesn't matter what order routes are defined.  Additionally, literal segments use simple hash table lookups.  What determines performance is the length of a path and the number of variable segments.

The matching algorithm goes as follows:

1. Incoming request data is passed to `TrieRouteMatcher::matchRoute()`, which loops through each segment of the URI path and proceeds only if there is either a literal or variable match in the URI tree
   * If there's a match, then we scan all child nodes against the next segment of the URI path and repeat step 1 until we don't find a match or we've matched the entire URI path
   * `TrieRouteMatcher::matchRoute()` uses <a href="http://php.net/manual/en/language.generators.syntax.php" target="_blank">generators</a> so we only descend the URI tree as many times as we need to find a match candidate
2. If the match candidate passes constraint checks (eg HTTP method constraints), then it's our matching route, and we're done.  Otherwise, repeat step 1, which will yield the next possible match candidate.

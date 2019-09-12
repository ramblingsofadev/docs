<h1 id="doc-title">Routing</h1>

<section class="toc">
<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
   1. [Route Variables](#route-variables)
   2. [Optional Route Parts](#optional-route-parts)
   3. [Route Builders](#route-builders)
   4. [Route Annotations](#route-annotations)
      1. [Example](#route-annotation-example)
      2. [Route Groups](#route-annotation-groups)
      3. [Middleware](#route-annotation-middleware)
      4. [Scanning For Annotations](#scanning-for-annotations)
   5. [Using Aphiria's Net Library](#using-aphirias-net-library)
   6. [Using Aphiria's Configuration Library](#using-aphirias-configuration-library)
2. [Route Actions](#route-actions)
3. [Binding Middleware](#binding-middleware)
   1. [Middleware Attributes](#middleware-attributes)
4. [Grouping Routes](#grouping-routes)
5. [Custom Constraints](#custom-constraints)
   1. [Example - Versioned API](#versioned-api-example)
   2. [Getting Headers in PHP](#getting-php-headers)
6. [Route Variable Rules](#route-variable-rules)
   1. [Built-In Rules](#built-in-rules)
   2. [Making Your Own Custom Rules](#making-your-own-custom-rules)
7. [Caching](#caching)
   1. [Route Caching](#route-caching)
   2. [Trie Caching](#trie-caching)
8. [Matching Algorithm](#matching-algorithm)
</section>

<h2 id="basics">Basics</h2>

This library is a routing library.  In other words, it lets you map URIs to actions, and attempts to match an input request to ones of those routes.

There are so many routing libraries out there.  Why use this one?  Well, there are a few reasons:

* It isn't coupled to _any_ library/framework
* It supports things that other route matching libraries do not support, like:
  * [Binding framework-agnostic middleware to routes](#binding-middleware)
  * [The ability to add custom matching rules on route variables](#route-variable-rules)
  * [The ability to match on header values](#custom-constraints), which makes things like versioning your routes a cinch
  * [Binding controller methods and closures to the route action](#route-actions)
* It is fast
  * With 400 routes, it takes ~0.0025ms to match any route (**~200% faster than FastRoute**)
  * The speed is due to the unique [trie-based matching algorithm](#matching-algorithm)
* Its [fluent syntax](#route-builders) keeps you from having to memorize how to set up config arrays
* It is built to support the latest PHP 7.4 features

Out of the box, this library provides a fluent syntax to help you build your routes.  Let's look at a working example.

First, let's import the namespaces and define our routes:

```php
use Aphiria\Routing\Builders\RouteBuilderRegistry;
use Aphiria\Routing\LazyRouteFactory;
use Aphiria\Routing\Matchers\Trees\{TrieFactory, TrieRouteMatcher};

// Register the routes
$routeFactory = new LazyRouteFactory(function () {
    $routes = new RouteBuilderRegistry();
    $routes->get('/books/:bookId')
        ->toMethod(BookController::class, 'getBooksById')
        ->withMiddleware(AuthMiddleware::class);
        
    return $routes->buildAll();
});

// Set up the route matcher
$routeMatcher = new TrieRouteMatcher((new TrieFactory($routeFactory))->createTrie());

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

If you'd like to use [rules](#route-variable-rules), then put them in parentheses after the variable:
```php
:varName(rule1,rule2(param1,param2))
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

Route builders give you a fluent syntax for mapping your routes to closures or controller methods.  They also let you [bind any middleware](#binding-middleware) classes and properties to the route.  The following methods are available to create routes:
  
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
  
 You can also call `RouteBuilderRegistry::map()` and pass in the HTTP method(s) you'd like to map to.

<h3 id="route-annotations">Route Annotations</h3>

Although annotations are not built into PHP, it is possible to include them in PHPDoc comments.  Aphiria provides the functionality to define your routes via PHPDoc annotations if you so choose.  A benefit to defining your routes this way is that it keeps the definition of your routes close (literally) to your controller methods, reducing the need to jump around your code base.

To use route annotations, install the route-annotations library by including the following in your _composer.json_:

```bash
"aphiria/route-annotations": "1.0.*"
```

> **Note:** Some IDEs <a href="https://www.doctrine-project.org/projects/doctrine-annotations/en/latest/index.html#ide-support" target="_blank">have plugins</a> that enable intellisense for PHPDoc annotations.

<h4 id="route-annotation-example">Example</h4>

Let's actually define a route:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Net\Http\IHttpResponseMessage;
use Aphiria\RouteAnnotations\Annotations\Put;
use App\Users\Http\Middleware\Authorization;
use App\Users\User;

class UserController extends Controller
{
    /**
     * @Put("users/:id")
     * @Middleware(Authorization::class)
     */
    public function updateUser(User $user): IHttpResponseMessage
    {
        // ...
    }
}
```

> **Note:** Controllers must either extend `Aphiria\Api\Controllers\Controller` or use the `@Controller` annotation.

The following HTTP methods have route annotations:

* `@Any` - Permits any HTTP method
* `@Delete`
* `@Get`
* `@Head`
* `@Options`
* `@Patch`
* `@Post`
* `@Put`
* `@Trace`

Each of the above annotations take in the route path as the first parameter, and optionally let you define any of the following properties:

* `host` - The host name for the route
* `name` - The name of the route
* `isHttpsOnly` - Whether or not the route is HTTPS only
* `attributes` - The key-value pairs of route metadata
* `constraints` - The list of `@RouteConstraint` options to apply
  * `@RouteConstraint` takes in the name of the constraint class and `constructorParams`, which is the list of parameters to pass into the constraint constructor

<h4 id="route-annotation-groups">Route Groups</h4>

Just like with our [route builders](#grouping-routes), we can also group routes with annotations:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Net\Http\IHttpResponseMessage;
use Aphiria\RouteAnnotations\Annotations\RouteGroup;
use App\Users\Http\Middleware\Authorization;
use App\Users\User;

/**
 * @RouteGroup("users")
 */
class UserController extends Controller
{
    /**
     * @Put(":id")
     * @Middleware(Authorization::class)
     */
    public function createUser(User $user): IHttpResponseMessage
    {
        // ...
    }
}
```

When our routes get compiled, the route group path will be prefixed to the path of any route within the controller.  In the above example, this would create a route with path `users/:id`.

The following properties can be set in `@RouteGroup`:

* `host` - The host name to suffix to the host of any routes contained in the controller
* `isHttpsOnly` - Whether or not all the routes are HTTPS only
* `attributes` - The key-value pairs of metadata that applies to all routes
* `constraints` - The list of `@RouteConstraint` options to apply to all routes
  
<h4 id="route-annotation-middleware">Middleware</h4>

Middleware can be defined via the `@Middleware` attribute.  The first value must be the name of the middleware class, and the following option is available:

* `attributes` - The key-value pairs of metadata for the middleware

<h4 id="scanning-for-annotations">Scanning For Annotations</h4>

Before you can use annotations, you'll need to configure Aphiria to scan for them.  The [configuration](configuring.md) library provides a convenience method for this:

```php
use Aphiria\Configuration\AphiriaComponentBuilder;
use Aphiria\RouteAnnotations\IRouteAnnotationRegistrant;
use Aphiria\RouteAnnotations\ReflectionRouteAnnotationRegistrant;

// Assume we already have $container set up
$routeAnnotationRegistrant = new ReflectionRouteAnnotationRegistrant(['PATH_TO_SCAN']);
$container->bindInstance(IRouteAnnotationRegistrant::class, $routeAnnotationRegistrant);

(new AphiriaComponentBuilder($container))
    ->withRoutingComponent($appBuilder)
    ->withRouteAnnotations($appBuilder);
```

If you're not using the configuration library, you can manually configure the router to scan for annotations:

```php
use Aphiria\RouteAnnotations\ReflectionRouteAnnotationRegistrant;
use Aphiria\Routing\Builders\RouteBuilderRegistry;
use Aphiria\Routing\LazyRouteFactory;
use Aphiria\Routing\Matchers\Trees\{TrieFactory, TrieRouteMatcher};

$routeFactory = new LazyRouteFactory(function () {
    $routeAnnotationRegistrant = new ReflectionRouteAnnotationRegistrant(['PATH_TO_SCAN']);
    $routes = new RouteBuilderRegistry();
    $routeAnnotationRegistrant->registerRoutes($routes);
    
    return $routes->buildAll();
});
$routeMatcher = new TrieRouteMatcher((new TrieFactory($routeFactory))->createTrie());

// Find a matching route
$result = $routeMatcher->matchRoute(
    $_SERVER['REQUEST_METHOD'],
    $_SERVER['HTTP_HOST'],
    $_SERVER['REQUEST_URI']
);
```

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

<h3 id="using-aphirias-configuration-library">Using Aphiria's Configuration Library</h3>

[Aphiria's configuration library](configuring.md) simplifies how you register routes.  Refer to [its documentation](configuring.md#configuring-routes) for more info.

<h2 id="route-actions">Route Actions</h2>

Aphiria supports mapping routes to both controller methods and to closures:

```php
// Map to a controller method
$routes->get('users/:userId')
    ->toMethod(UserController::class, 'getUserById');

// Map to a closure
$routes->get('users/:userId/name')
    ->toClosure(function () {
        // Handle the request...
    });
```

To determine the type of action (controller method or closure) the matched route uses, check `RouteAction::usesMethod()`.

<h2 id="binding-middleware">Binding Middleware</h2>

Middleware are a great way to modify both the request and the response on an endpoint.  Aphiria lets you define middleware on your endpoints without binding you to any particular library/framework's middleware implementations.

To bind a single middleware class to your route, call:

```php
$routes->get('foo')
    ->toMethod(MyController::class, 'myMethod')
    ->withMiddleware(FooMiddleware::class);
```

To bind many middleware classes, call:

```php
$routes->get('foo')
    ->toMethod(MyController::class, 'myMethod')
    ->withManyMiddleware([
        FooMiddleware::class,
        BarMiddleware::class
    ]);
```

Under the hood, these class names get converted to instances of `MiddlewareBinding`.  

<h3 id="middleware-attributes">Middleware Attributes</h3>

Some frameworks, such as Aphiria and Laravel, let you bind attributes to middleware.  For example, if you have an `AuthMiddleware`, but need to bind the user role that's necessary to access that route, you might want to pass in the required user role.  Here's how you can do it:

```php
$routes->get('foo')
    ->toMethod(MyController::class, 'myMethod')
    ->withMiddleware(AuthMiddleware::class, ['role' => 'admin']);

// Or

$routes->get('foo')
    ->toMethod(MyController::class, 'myMethod')
    ->withManyMiddleware([
        new MiddlewareBinding(AuthMiddleware::class, ['role' => 'admin']),
        // Other middleware...
    ]);
```

Here's how you can grab the middleware on a matched route:

```php
foreach ($result->middlewareBindings as $middlewareBinding) {
    $middlewareBinding->className; // "AuthMiddleware"
    $middlewareBinding->attributes; // ["role" => "admin"]
}
```

<h2 id="grouping-routes">Grouping Routes</h2>

Often times, a lot of your routes will share similar properties, such as hosts and paths to match on, or middleware.  You can group these routes together using `RouteBuilderRegistry::group()` and specifying the options to apply to all routes within the group:

```php
use Aphiria\Routing\Builders\RouteGroupOptions;

$routes->group(
    new RouteGroupOptions('courses/:courseId', 'example.com'),
    function (RouteBuilderRegistry $routes) {
        // This route's path will use the group's path
        $routes->get('')
            ->toMethod(CourseController::class, 'getCourseById');

        $routes->get('/professors')
            ->toMethod(CourseController::class, 'getCourseProfessors');
    }
);
```

This creates two routes with a host suffix of _example.com_ and a route prefix of _users/_ (`example.com/courses/:courseId` and `example.com/courses/:courseId/professors`).  `RouteGroupOptions::__construct()` accepts the following parameters:

* `string $pathTemplate`
  * The path for routes in this group ([read about syntax](#route-variables))
  * This value is prefixed to the paths of all routes within the group
* `string|null $hostTemplate` (optional)
  * The optional host template for routes in this group  ([read about syntax](#route-variables))
  * This value is suffixed to the hosts of all routes within the group
* `bool $isHttpsOnly` (optional)
  * Whether or not the routes in this group are HTTPS-only
* `IRouteConstraint[] $constraints` (optional)
* `array $attributes` (optional)
  * The mapping of route attribute names => values
  * These attribute can be used with [custom constraint](#custom-constraints) matching
* `MiddlewareBinding[] $middleware` (optional)
  * The list of middleware bindings for routes in this group

It is possible to nest route groups.

<h2 id="custom-constraints">Custom Constraints</h2>

Sometimes, you might find it useful to add some custom logic for matching routes.  This could involve enforcing anything from only allowing certain HTTP methods for a route (eg `HttpMethodRouteConstraint`) or only allowing HTTPS requests to a particular endpoint.  Let's go into some concrete examples...

<h3 id="versioned-api-example">Example - Versioned API</h3>

Let's say your app sends an API version header, and you want to match an endpoint that supports that version.  You could do this by using a route "attribute" and a route constraint.  Let's create some routes that have the same path, but support different versions of the API:

```php
// This route will require an API-VERSION value of 'v1.0'
$routes->get('comments')
    ->toMethod(CommentController::class, 'getAllComments1_0')
    ->withAttribute('API-VERSION', 'v1.0')
    ->withConstraint(new ApiVersionConstraint);

// This route will require an API-VERSION value of 'v2.0'
$routes->get('comments')
    ->toMethod(CommentController::class, 'getAllComments2_0')
    ->withAttribute('API-VERSION', 'v2.0')
    ->withConstraint(new ApiVersionConstraint);
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
        string $host,
        string $path,
        array $headers
    ): bool {
        $attributes = $matchedRouteCandidate->route->attributes;

        if (!isset($attributes['API-VERSION'])) {
            return false;
        }

        return array_search($attributes['API-VERSION'], $headers['API-VERSION']) !== false;
    }
}
```

If we hit _/comments_ with an "API-VERSION" header value of "v2.0", we'd match the second route in our example.

<h3 id="getting-php-headers">Getting Headers in PHP</h3>

PHP is irritatingly difficult to extract headers from `$_SERVER`.  If you're using a library/framework to grab headers, then use that.  Otherwise, you can use the `HeaderParser`:

```php
use Aphiria\Routing\Requests\HeaderParser;

$headers = (new HeaderParser)->parseHeaders($_SERVER);
```

<h2 id="route-variable-rules">Route Variable Rules</h2>

You can enforce certain rules to pass before matching on a route.  These rules come after variables, and must be enclosed in parentheses.  For example, if you want an integer to fall between two values, you can specify a route of

```php
:month(int,min(1),max(12))
```

> **Note:** If a rule does not require any parameters, then the parentheses after the rule slug are optional.

<h3 id="built-in-rules">Built-In Rules</h3>

The following rules are built-into Aphiria:

* `alpha`
* `alphanumeric`
* `between($min, $max, bool $isInclusive = true)`
* `date(string $commaSeparatedListOfAcceptableFormats)`
* `in(string $commaSeparatedListOfAcceptableValues)`
* `int`
* `notIn(string $commaSeparatedListOfUnacceptableValues)`
* `numeric`
* `regex(string $regex)`
* `uuidv4`

<h3 id="making-your-own-custom-rules">Making Your Own Custom Rules</h3>

You can register your own rule by implementing `IRule`.  Let's make a rule that enforces a certain minimum string length:

```php
use Aphiria\Routing\Matchers\Rules\IRule;

final class MinLengthRule implements IRule
{
    private int $minLength;

    public function __construct(int $minLength)
    {
        $this->minLength = $minLength;
    }

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

Let's register our rule with the rule factory:

```php
use Aphiria\Routing\Matchers\Rules\{RuleFactory, RuleFactoryRegistrant};

// Register some built-in rules to our factory
$ruleFactory = (new RuleFactoryRegistrant)->registerRuleFactories(new RuleFactory);

// Register our custom rule
$ruleFactory->registerRuleFactory(MinLengthRule::getSlug(), function (int $minLength) {
    return new MinLengthRule($minLength);
});
```

Finally, register this rule factory with the trie compiler:

```php
use Aphiria\Routing\Builders\RouteBuilderRegistry;
use Aphiria\Routing\LazyRouteFactory;
use Aphiria\Routing\Matchers\Trees\{TrieFactory, TrieRouteMatcher};

$routeFactory = new LazyRouteFactory(function () {
    $routes = new RouteBuilderRegistry();
    $routes->get('parts/:serialNumber(minLength(6))')
        ->toMethod(PartController::class, 'getPartBySerialNumber');
        
    return $routes->buildAll();
});

$trieCompiler = new TrieCompiler($ruleFactory);
$trieFactory = new TrieFactory($routeFactory, null, $trieCompiler);
$routeMatcher = new TrieRouteMatcher($trieFactory->createTrie());
```

Our route will now enforce a serial number with minimum length 6.

<h2 id="caching">Caching</h2>

The process of building your routes and compiling the trie is a relatively slow process, and isn't necessary in a production environment where route definitions aren't changing.  Aphiria provides both the ability to cache the results of your route builders and the compiled trie.

<h3 id="route-caching">Route Caching</h3>

To enable caching, pass in an `IRouteCache` (`FileRouteCache` is provided) to the second parameter of `LazyRouteFactory`:

```php
use Aphiria\Routing\Caching\FileRouteCache;
use Aphiria\Routing\LazyRouteFactory;

$routeFactory = new LazyRouteFactory(
    function () {
        // Register your routes...
    },
    new FileRouteCache('/tmp/routes.cache')
);

// Finish setting up your route matcher...
```

To enable caching in a particular environment, you could do this:

```php
$routeFactory = new LazyRouteFactory(
    function () {
        // Register your routes...
    },
    getenv('ENV_NAME') === 'production' ? new FileRouteCache('/tmp/trie.cache') : null;
);
```

<h3 id="trie-caching">Trie Caching</h3>

To enable caching, pass in an `ITrieCache` (`FileTrieCache` comes with Aphiria) to your trie factory (passing in `null` will disable caching).  If you want to enable caching for a particular environment, you could do so:

```php
use Aphiria\Routing\Matchers\Trees\Caching\FileTrieCache;
use Aphiria\Routing\Matchers\Trees\TrieFactory;

// Let's say that your environment name is stored in an environment var named 'ENV_NAME'
$trieCache = getenv('ENV_NAME') === 'production' ? new FileTrieCache('/tmp/trie.cache') : null;
$trieFactory = new TrieFactory($routeFactory, $trieCache);

// Finish setting up your route matcher...
```

<h2 id="matching-algorithm">Matching Algorithm</h2>

Rather than the typical regex approach to route matching, we decided to go with a <a href="https://en.wikipedia.org/wiki/Trie" target="_blank">trie-based</a> approach.  Each node maps to a segment in the path, and could either contain a literal or a variable value.  We try to proceed down the tree to match what's in the request URI, always giving preference to literal matches over variable ones, even if variable segments are declared first in the routing config.  This logic not only applies to the first segment, but recursively to all subsequent segments.  The benefit to this approach is that it doesn't matter what order routes are defined.  Additionally, literal segments use simple hash table lookups.  What determines performance is the length of a path and the number of variable segments.

The matching algorithm goes as follows:

1. Incoming request data is passed to `TrieRouteMatcher::matchRoute()`, which loops through each segment of the URI path and proceeds only if there is either a literal or variable match in the URI tree
   * If there's a match, then we scan all child nodes against the next segment of the URI path and repeat step 1 until we don't find a match or we've matched the entire URI path
   * `TrieRouteMatcher::matchRoute()` uses <a href="http://php.net/manual/en/language.generators.syntax.php" target="_blank">generators</a> so we only descend the URI tree as many times as we need to find a match candidate
2. If the match candidate passes constraint checks (eg HTTP method constraints), then it's our matching route, and we're done.  Otherwise, repeat step 1, which will yield the next possible match candidate.

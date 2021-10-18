<h1 id="doc-title">Routing</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
   1. [Route Variables](#route-variables)
   2. [Optional Route Parts](#optional-route-parts)
   3. [Route Groups](#route-groups)
   4. [Middleware](#middleware)
   5. [Route Constraints](#route-constraints)
2. [Route Attributes](#route-attributes)
   1. [Example](#route-attributes-example)
   2. [Route Groups](#route-attributes-groups)
   3. [Middleware](#route-attributes-middleware)
   4. [Route Constraints](#route-attributes-constraints)
   5. [Scanning For Attributes](#scanning-for-attributes)
3. [Route Builders](#route-builders)
   1. [Route Groups](#route-builders-groups)
   2. [Middleware](#route-builders-middleware)
   3. [Route Constraints](#route-builders-constraints)
4. [Versioned API Example](#versioned-api-example)
   1. [Getting Headers in PHP](#getting-php-headers)
5. [Route Variable Constraints](#route-variable-constraints)
   1. [Built-In Constraints](#built-in-constraints)
   2. [Making Your Own Custom Constraints](#making-your-own-custom-constraints)
6. [Creating Route URIs](#creating-route-uris)
7. [Caching](#caching)
   1. [Route Caching](#route-caching)
   2. [Trie Caching](#trie-caching)
8. [Using Aphiria's Net Library](#using-aphirias-net-library)
9. [Matching Algorithm](#matching-algorithm)

</div>

</nav>

<h2 id="basics">Basics</h2>

Routing is the process of mapping HTTP requests to actions.  You can check out what makes Aphiria's routing library different [here](framework-comparisons.md#aphiria-routing) as well as the [server configuration](installation.md#server-config) necessary to use it.

Let's look at a fully-functional example (or view its [attribute-based alternative](#route-attributes-example)):

```php
use Aphiria\Routing\Builders\RouteCollectionBuilder;
use Aphiria\Routing\Matchers\TrieRouteMatcher;
use Aphiria\Routing\UriTemplates\Compilers\Tries\TrieFactory;
use App\Books\Api\{Authorization, BookController};

// Register the routes
$routes = new RouteCollectionBuilder();
$routes->get('/books/:bookId')
    ->mapsToMethod(BookController::class, 'getBookById')
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

To get the [middleware bindings](#middleware), call:

```php
foreach ($result->route->middlewareBindings as $middlewareBinding) {
    $middlewareBinding->className; // "Authorization"
    $middlewareBinding->parameters; // ["role" => "admin"]
}
```

If you're using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a> and `$result->methodIsAllowed`, a 405 response will automatically be returned with a list of allowed methods.  If you're not, you can manually do the same thing:

```php
header('Allow', implode(', ', $result->allowedMethods));
```

<h3 id="route-variables">Route Variables</h3>

Aphiria provides a simple syntax for your URIs.  To capture variables in your route, use `:varName`, eg:

```php
users/:userId/profile
```

You can also add [constraints](#route-variable-constraints) to your variables.

<h3 id="optional-route-parts">Optional Route Parts</h3>

If part of your route is optional, then surround it with brackets.  For example, the following will match both `archives/2017` and `archives/2017/7`:
```php
archives/:year[/:month]
```

Optional route parts can be nested:

```php
archives/:year[/:month[/:day]]
```

This would match `archives/2017`, `archives/2017/07`, and `archives/2017/07/24`.

<h3 id="route-groups">Route Groups</h3>

Often times, a lot of your routes will share similar properties, such as hosts and paths to match on, or middleware.  Route groups can even be nested.  Learn more how to add them as [attributes](#route-attributes-groups) or [route builders](#route-builders-groups).

<h3 id="middleware">Middleware</h3>

Middleware are a great way to modify both the request and the response on an endpoint.  Aphiria lets you define middleware on your endpoints without binding you to any particular library/framework's middleware implementations.  Learn how to add them as [attributes](#route-attributes-middleware) or [route buidlers](#route-builders-middleware).

<h4 id="middleware-parameters">Middleware Parameters</h4>

Some frameworks, such as Aphiria and Laravel, let you bind parameters to middleware.  For example, if you have an `Authorization` middleware, but need to bind the user role that's necessary to access that route, you might want to pass in the required user role.  Learn more about how to specify middleware parameters as [attributes](#route-attributes-middleware) or [route builders](#route-builders-middleware).

<h3 id="route-constraints">Route Constraints</h3>

Sometimes, you might find it useful to add some custom logic for matching routes.  This could involve enforcing anything from only allowing certain HTTP methods for a route (eg `HttpMethodRouteConstraint`) or only allowing HTTPS requests to a particular endpoint.  Learn more how to add them as [attributes](#route-attributes-constraints) or [route builders](#route-builders-constraints).

<h2 id="route-attributes">Route Attributes</h2>

Aphiria provides the optional functionality to define your routes via attributes if you so choose.  A benefit to defining your routes this way is that it keeps the definition of your routes close (literally) to your controller methods, reducing the need to jump around your code base.

<h3 id="route-attributes-example">Example</h3>

Let's actually define a route:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Routing\Attributes\{Get, Middleware};
use App\Books\Api\Authorization;
use App\Books\Book;

final class BookController extends Controller
{
    #[Get('/books/:bookId'), Middleware(Authorization::class)]
    public function getBookById(int $bookId): Book
    {
        // ...
    }
}
```

> **Note:** Controllers must either extend `Aphiria\Api\Controllers\Controller` or use the `#[Controller]` attribute.

The following HTTP methods have route attributes:

* `#[Any]` (any HTTP method)
* `#[Delete]`
* `#[Get]`
* `#[Head]`
* `#[Options]`
* `#[Patch]`
* `#[Post]`
* `#[Put]`
* `#[Trace]`

Each attribute takes in the same parameters:

```php
use Aphiria\Routing\Attributes\Get;

#[Get(
    path: 'courses/:courseId',
    host: 'api.example.com',
    name: 'getCourse',
    isHttpsOnly: true,
    parameters: ['role' => 'admin']
)]
```

<h3 id="route-attributes-groups">Route Groups</h3>

You can apply route groups, constraints, and middleware to all endpoints in a controller:

```php
use Aphiria\Api\Controllers\Controller;
use Aphiria\Routing\Attributes\{Get, Middleware, RouteConstraint, RouteGroup};
use App\Courses\Api\Authorization;
use App\Courses\Course;

#[
    RouteGroup(
        path: 'courses/:courseId',
        host: 'api.example.com',
        isHttpsOnly: true,
        parameters: ['role' => 'admin']
    ),
    RouteConstraint(MyConstraint::class),
    Middleware(Authentication::class)
]
final class CourseController extends Controller
{
    #[Get('')]
    public function getCourseById(int $courseId): Course
    {
        // ...
    }
    
    #[Get('professors')]
    public function getCourseProfessors(): array
    {
        // ...
    }
}
```

When our routes get compiled, the route group path will be prefixed to the path of any route within the controller.  In the above example, this would create a route with path `courses/:courseId` and another with path `courses/:courseId/professors`.
  
<h3 id="route-attributes-middleware">Middleware</h3>

Middleware are a separate attribute:

```php
use Aphiria\Routing\Attributes\Middleware;

#[Middleware(Authorization::class, parameters: ['role' => 'admin'])]
```

You can also add middleware to a controller class to indicate that it applies to all routes in that controller.

<h3 id="route-attributes-constraints">Route Constraints</h3>

You can specify the name of the route constraint class and any primitive constructor parameter values:

```php
use Aphiria\Routing\Attributes\{Get, RouteConstraint};
use App\Users\User;

final class UserController extends Controller
{
    #[
        Get('users/:userId'),
        RouteConstraint(MyConstraint::class, constructorParameters: ['param1'])
    ]
    public function getUserById(int $userId): User
    {
        // ...
    }
}
```

Similar to [middleware](#route-attributes-middleware), you can add route constraints to a controller class to apply it to all routes in that controller.

<h3 id="scanning-for-attributes">Scanning For Attributes</h3>

Before you can use attributes, you'll need to configure Aphiria to scan for them.  If you're using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, you can do so in `GlobalModule`:

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Framework\Application\AphiriaComponents;

final class GlobalModule implements IModule
{
    use AphiriaComponents;

    public function configure(IApplicationBuilder $appBuilder): void
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

<h2 id="route-builders">Route Builders</h2>

Route builders are an alternative to [attributes](#route-attributes) that give you a fluent syntax for mapping your routes to controller methods.  They also let you [bind any middleware](#route-builders-middleware) classes and properties to the route.  The following methods are available to create routes:

 ```php
$routes->delete('/foo');
$routes->get('/foo');
$routes->options('/foo');
$routes->patch('/foo');
$routes->post('/foo');
$routes->put('/foo');
```

Each method accepts the following parameters:

```php
$routes->get(path: '/user', host: 'api.example.com', isHttpsOnly: true)
    ->mapsToMethod(UserController::class, 'getUserById');
```

They all return an instance of `RouteBuilder`, which lets you specify things like controller methods, [middleware](#route-builders-middleware), and [constraints](#route-builders-constraints).

You can also call `RouteCollectionBuilder::route()` and pass in the HTTP method(s) you'd like to map to.

```php
$routes->route(['GET'], path: '/user', host: 'api.example.com', isHttpsOnly: true);
```

If you're using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, the best place to define your routes is in modules using [application builders](configuration.md#component-routes).

<h3 id="route-builders-groups">Route Groups</h3>

```php
use Aphiria\Routing\Builders\RouteCollectionBuilder;
use Aphiria\Routing\Builders\RouteGroupOptions;
use Aphiria\Routing\Middleware\MiddlewareBinding;

$routes->group(
    new RouteGroupOptions(
        path: 'courses/:courseId',
        host: 'api.example.com',
        isHttpsOnly: true,
        constraints: [new MyConstraint()],
        middlewareBindings: [new MiddlewareBinding(Authentication::class)],
        parameters: ['role' => 'admin']
    ),
    function (RouteCollectionBuilder $routes) {
        // This route's path will be 'courses/:courseId'
        $routes->get('')
            ->mapsToMethod(CourseController::class, 'getCourseById');

        // This route's path will be 'courses/:courseId/professors'
        $routes->get('/professors')
            ->mapsToMethod(CourseController::class, 'getCourseProfessors');
    }
)
```

<h3 id="route-builders-middleware">Middleware</h3>

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

You can also add [parameters to your middleware](#middleware-parameters):

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

<h3 id="route-builders-constraints">Route Constraints</h3>

To add a single route constraint to a route, call:

```php
$routes->get('posts')
    ->mapsToMethod(PostController::class, 'getAllPosts')
    ->withConstraint(new FooConstraint());
```

To add many route constraints, call:

```php
$routes->get('posts')
    ->mapsToMethod(PostController::class, 'getAllPosts')
    ->withManyConstraints([new FooConstraint(), new BarConstraint()]);
```

<h2 id="versioned-api-example">Versioned API Example</h2>

Let's say your app sends an API version header, and you want to match an endpoint that supports that version.  You could do this by using a route "parameter" and a route constraint.  Let's create some routes that have the same path, but support different versions of the API:

```php
use Aphiria\Routing\Attributes\{Get, RouteConstraint};

final class CommentController extends Controller
{
    #[
        Get('comments', parameters: ['API-VERSION' => 'v1.0']),
        RouteConstraint(ApiVersionConstraint::class)
    ]
    public function getAllComments1_0(): array
    {
        // This route will require an API-VERSION value of 'v1.0'
    }
    
    #[
        Get('comments', parameters: ['API-VERSION' => 'v2.0']),
        RouteConstraint(ApiVersionConstraint::class)
    ]
    public function getAllComments2_0(): array
    {
        // This route will require an API-VERSION value of 'v2.0'
    }
}
```

Now, let's add a route constraint to match the "API-VERSION" header to the parameter on our route:

```php
use Aphiria\Routing\Matchers\Constraints\IRouteConstraint;
use Aphiria\Routing\Matchers\MatchedRouteCandidate;

final class ApiVersionConstraint implements IRouteConstraint
{
    public function passes(
        MatchedRouteCandidate $matchedRouteCandidate,
        string $httpMethod,
        string $host,
        string $path,
        array $headers
    ): bool {
        $parameters = $matchedRouteCandidate->route->parameters;

        if (!isset($parameters['API-VERSION'])) {
            return false;
        }

        return \in_array($parameters['API-VERSION'], $headers['API-VERSION'], true);
    }
}
```

If we hit `/comments` with an "API-VERSION" header value of "v2.0", we'd match the second route in our example.

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
    public function __construct(private int $minLength) {}

    public static function getSlug(): string
    {
        return 'minLength';
    }

    public function passes($value): bool
    {
        return \mb_strlen($value) >= $this->minLength;
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

You might find yourself wanting to create a link to a particular route within your app.  Let's say you have a route named `GetUserById` with a URI template of `/users/:userId`.  We can generate a link to get a particular user:

```php
use Aphiria\Routing\UriTemplates\AstRouteUriFactory;

// Assume you've already created your routes
$routeUriFactory = new AstRouteUriFactory($routes);

// Will create "/users/123"
$uriForUser123 = $routeUriFactory->createRouteUri('GetUserById', ['id' => 123]);
```

Generated URIs will be a relative path unless the URI template specified a host.  Let's look at an example for one that does include a host: `:environment.example.com/users/:userId`.

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
$trieCache = \getenv('ENV_NAME') === 'production'
    ? new FileTrieCache('/tmp/trieCache.txt')
    : null;
$trieFactory = new TrieFactory($routes, $trieCache);

// Finish setting up your route matcher...
```

<h2 id="using-aphirias-net-library">Using Aphiria's Net Library</h2>

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

<h2 id="matching-algorithm">Matching Algorithm</h2>

Rather than the typical regex approach to route matching, we decided to go with a <a href="https://en.wikipedia.org/wiki/Trie" target="_blank">trie-based</a> approach.  Each node maps to a segment in the path, and could either contain a literal or a variable value.  We try to proceed down the tree to match what's in the request URI, always giving preference to literal matches over variable ones, even if variable segments are declared first in the routing config.  This logic not only applies to the first segment, but recursively to all subsequent segments.  The benefit to this approach is that it doesn't matter what order routes are defined.  Additionally, literal segments use simple hash table lookups.  What determines performance is the length of a path and the number of variable segments.

The matching algorithm goes as follows:

1. Incoming request data is passed to `TrieRouteMatcher::matchRoute()`, which loops through each segment of the URI path and proceeds only if there is either a literal or variable match in the URI tree
   * If there's a match, then we scan all child nodes against the next segment of the URI path and repeat step 1 until we don't find a match or we've matched the entire URI path
   * `TrieRouteMatcher::matchRoute()` uses <a href="http://php.net/manual/en/language.generators.syntax.php" target="_blank">generators</a> so we only descend the URI tree as many times as we need to find a match candidate
2. If the match candidate passes constraint checks (eg HTTP method constraints), then it's our matching route, and we're done.  Otherwise, repeat step 1, which will yield the next possible match candidate.

<h1 id="doc-title">HTTP Exception Handling</h1>

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
   1. [Creating Response For Exceptions](#creating-responses-for-exceptions)
2. [Handling Exceptions During Request Handling](#handling-exceptions-during-request-handling)
3. [Global Exception Handling](#global-exception-handling)
   1. [Throwing Errors As Exceptions](#throwing-errors-as-exceptions)
4. [Customizing Exception Responses](#customizing-exception-responses)
   1. [Using Classes to Create Exception Responses](#using-classes-to-create-exception-responses)
5. [Logging](#logging)

<h2 id="basics">Basics</h2>

Sometimes, your application is going to throw an unhandled exception or shut down unexpectedly.  When this happens, instead of showing an ugly PHP error, you can convert it to a nicely-formatted response.  You can configure Aphiria to [handle exceptions thrown while handling HTTP requests](#handling-exceptions-during-request-handling) via middleware as well as [handle any unhandled exceptions anywhere in your application lifespan](#global-exception-handling).  However, you are not obligated to use Aphiria's exception handlers - you may use any that you'd like.

The first thing we need to do is set up a factory that can create HTTP responses from exceptions using [content negotiation](content-negotiation.md):

```php
use Aphiria\Exceptions\ExceptionResponseFactory;

$exceptionResponseFactory = new ExceptionResponseFactory();
```

Now, let's start handling some exceptions.

<h2 id="handling-exceptions-during-request-handling">Handling Exceptions During Request Handling</h2>

We can use middleware to catch any exceptions that might be thrown while handling a request.  Let's look at an example that uses the [configuration library](configuring.md):

```php
use Aphiria\Exceptions\ExceptionLogger;
use Aphiria\Exceptions\IExceptionLogger;
use Aphiria\Exceptions\Middleware\ExceptionHandler;

// Assume our container and application builder are already set up
$container->bindInstance(IExceptionLogger::class, new ExceptionLogger());
$container->bindInstance(IExceptionResponseFactory::class, $exceptionResponseFactory);

// ...

$appBuilder->withGlobalMiddleware(fn () => [
    new MiddlewareBinding(ExceptionHandler::class)
]);

// ...
```

Now, whenever an exception is thrown later on down the middleware pipeline, our `ExceptionHandler` middleware will catch it, [log it](#logging), and return a response.

<h2 id="global-exception-handling">Global Exception Handling</h2>

Sometimes, exceptions or PHP errors can be thrown before we even get to the middleware pipeline.  In this case, it's a good idea to set up a global exception handler with PHP.

```php
use Aphiria\Exceptions\ExceptionLogger;
use Aphiria\Exceptions\GlobalExceptionHandler;

$globalExceptionHandler = new GlobalExceptionHandler(
    $exceptionResponseFactory,
    new ExceptionLogger()
);

// Let PHP know to use this as our exception and error handler:
$globalExceptionHandler->registerWithPhp();
```

<h3 id="throwing-errors-as-exceptions">Throwing Errors As Exceptions</h3>

You can configure the global exception handler to rethrow PHP errors as an `ErrorException` by specifying a bit-wise list of error levels:

```php
$globalExceptionHandler = new GlobalExceptionHandler(
    $exceptionResponseFactory,
    null,
    E_ALL & ~(E_DEPRECATED | E_USER_DEPRECATED)
);
```

<h2 id="customizing-exception-responses">Customizing Exception Responses</h2>

You might find yourself wanting to map a particular exception to a certain response.  In this case, you can use an exception response factory.  They are closures that take in the exception and the request, and return a response.

As an example, let's say that you want to return a 404 response when an `EntityNotFound` exception is thrown:

```php
use Aphiria\Exceptions\{ExceptionResponseFactory, ExceptionResponseFactoryRegistry};

// Register your custom exception response factories
$exceptionResponseFactories = new ExceptionResponseFactoryRegistry();
$exceptionResponseFactories->registerFactory(
    EntityNotFound::class,
    fn (EntityNotFound $ex, ?IHttpRequestMessage $request, INegotiatedResponseFactory $responseFactory) =>
        $responseFactory->createResponse($request, 404, null, null)
);

// Assume the content negotiator was already set up
$exceptionResponseFactory = new ExceptionResponseFactory(
    new NegotiatedResponseFactory($contentNegotiator),
    $exceptionResponseFactories
);

// Pass the factory to your middleware and global exception handler...
```

That's it.  Now, whenever an unhandled `EntityNotFound` exception is thrown, your application will return a 404 response that uses [content negotiation](content-negotiation.md).  You can also register multiple exception factories at once.  Just pass in an array, keyed by exception type:

```php
$exceptionResponseFactories->registerManyFactories([
    EntityNotFound::class => fn (EntityNotFound $ex, ?IHttpRequestMessage $request, INegotiatedResponseFactory $responseFactory)
        $responseFactory->createResponse($request, 404, null, null),
    // ...
]);
```

<h3 id="using-classes-to-create-exception-responses">Using Classes to Create Exception Responses</h3>

Sometimes, the logic inside your exception response factory might get a little too complicated to be easily readable in a `Closure`.  In this case, you can also use a POPO to encapsulate your response creation logic:

```php
final class WhoopsResponseFactory
{
    public function createResponse(Exception $ex, ?IHttpRequestMessage $request): IHttpResponseMessage
    {
        $response = new Response();
        // Finish creating your response...
        
        return $response;
    }
}

$exceptionResponseFactories->registerFactory(
    Exception::class,
    fn (Exception $ex, ?IHttpRequestMessage $request, INegotiatedResponseFactory $responseFactory) 
        => (new WhoopsResponseFactory)->createResponse($ex, $request)
);
```

<h2 id="logging">Logging</h2>

`IExceptionLogger` contains methods to handle logging both PHP errors and exceptions.  `ExceptionLogger` is enabled by default, and uses <a href="https://github.com/Seldaek/monolog" target="_blank">Monolog</a> to do the actual logging.

```php
use Aphiria\Exceptions\ExceptionLogger;

$exceptionLogger = new ExceptionLogger();

// Inject the logger into your exception handlers...
```

You can specify your own custom logger that implements the PSR-3 `LoggerInterface`:

```php
use Aphiria\Exceptions\ExceptionResponseFactory;
use Monolog\Handler\ErrorLogHandler;
use Monolog\Logger;

$logger = new Logger('app');
$logger->pushHandler(new SyslogHandler());
$exceptionLogger = new ExceptionLogger($logger);
```

It's possible to specify some rules around the <a href="https://www.php-fig.org/psr/psr-3/#5-psrlogloglevel" target="_blank">PSR-3 log level</a> that an exception returns.  This could be useful for things like logging 500s as critical, but everything else as warnings.  Let's look at an example:

```php
use Aphiria\Exceptions\ExceptionLogLevelFactoryRegistry;

$exceptionLogLevelFactories = new ExceptionLogLevelFactoryRegistry();
$exceptionLogLevelFactories->registerFactory(
    HttpException::class,
    function (HttpException $ex) {
        if ($ex->getResponse()->getStatusCode() >= 500) {
            return LogLevel::CRITICAL;
        }
        
        return LogLevel::WARNING;
    }
);
$exceptionLogger = new ExceptionLogger(
    null,
    $exceptionLogLevelFactories
);

// Inject the logger into your exception handlers...
```

> **Note:** You can register many factories at once using `ExceptionLogLevelFactoryRegistry::registerManyFactories()`.

Passing in an array of PSR-3 log levels will cause only those levels to be logged:

```php
$exceptionLogger = new ExceptionLogger(
    null,
    null,
    [LogLevel::CRITICAL, LogLevel::EMERGENCY]
);
```

By default, `LogLevel::ERROR`, `LogLevel::CRITICAL`, `LogLevel::ALERT`, and `LogLevel::EMERGENCY` will be logged if `null` is specified.

You can also control the level of PHP errors that are logged by specifying a bitwise value similar to what's in your _php.ini_:

```php
$exceptionLogger = new ExceptionLogger(
    null, 
    null, 
    null,
    E_ALL & ~E_NOTICE
);
```

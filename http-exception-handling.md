# HTTP Exception Handling

## Table of Contents
1. [Basics](#basics)
2. [Customizing Exception Responses](#customizing-exception-responses)
  1. [Using Classes to Create Exception Responses](#using-classes-to-create-exception-responses)
3. [Logging](#logging)

<h1 id="basics">Basics</h1>

Sometimes, your application is going to throw an unhandled exception or shut down unexpectedly.  When this happens, instead of showing an ugly PHP error, you can convert it to a nicely-formatted response.  To get set up, you can simply instantiate `ExceptionHandler` and register it with PHP:

```php
use Aphiria\Api\Exceptions\{ExceptionHandler, ExceptionResponseFactory};
use Aphiria\Net\Http\ContentNegotiation\NegotiatedResponseFactory;

// Assume the content negotiator was already set up
$exceptionResponseFactory = new ExceptionResponseFactory(
    new NegotiatedResponseFactory($contentNegotiator)
);

$exceptionHandler = new ExceptionHandler($exceptionResponseFactory);
$exceptionHandler->registerWithPhp();
```

By default, `ExceptionHandler` will convert any exception to a 500 response and use [content negotiation](content-negotiation) to determine the best format for the response body.  However, you can [customize your exception responses](#customizing-exception-responses).

<h1 id="customizing-exception-responses">Customizing Exception Responses</h1>

You might find yourself wanting to map a particular exception to a certain response.  In this case, you can use an exception response factory.  They are closures that take in the exception and the request, and return a response.

As an example, let's say that you want to return a 404 response when an `EntityNotFound` exception is thrown:

```php
use Aphiria\Api\Exceptions\{ExceptionResponseFactory, ExceptionResponseFactoryRegistry};
use Aphiria\Net\Http\Response;

// Register your custom exception response factories
$exceptionResponseFactories = new ExceptionResponseFactoryRegistry();
$exceptionResponseFactories->registerFactory(
    EntityNotFound::class,
    fn (EntityNotFound $ex, ?IHttpRequestMessage $request) => new Response(HttpStatusCodes::HTTP_NOT_FOUND)
);

// Assume the content negotiator was already set up
$exceptionResponseFactory = new ExceptionResponseFactory(
    new NegotiatedResponseFactory($contentNegotiator),
    $exceptionResponseFactories
);

// Add it to the exception handler
$exceptionHandler = new ExceptionHandler($exceptionResponseFactory);
$exceptionHandler->registerWithPhp();
```

That's it.  Now, whenever an unhandled `EntityNotFound` exception is thrown, your application will return a 404 response.  You can also register multiple exception factories at once.  Just pass in an array, keyed by exception type:

```php
$exceptionResponseFactories->registerManyFactories([
    EntityNotFound::class => fn (EntityNotFound $ex, ?IHttpRequestMessage $request) => new Response(404),
    // ...
]);
```

If you want to take advantage of automatic content negotiation, you can use a `NegotiatedResponseFactory` in your factory:

```php
use Aphiria\Net\Http\ContentNegotiation\NegotiatedResponseFactory;

// Assume the content negotiator was already set up
$negotiatedResponseFactory = new NegotiatedResponseFactory($contentNegotiator);
// ...
$exceptionResponseFactories->registerFactory(
    EntityNotFound::class,
    function (EntityNotFound $ex, ?IHttpRequestMessage $request) use ($negotiatedResponseFactory) {
        $error = new MyErrorObject('Entity not found');
    
        return $negotiatedResponseFactory->createResponse($request, 404, null, $error);
    }
);
```

If an unhandled `EntityNotFound` exception was thrown, your exception factory will use content negotiation to serialize `MyErrorObject` in the response body.

<h2 id="using-classes-to-create-exception-responses">Using Classes to Create Exception Responses</h2>

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
    fn (Exception $ex, ?IHttpRequestMessage $request) => (new WhoopsResponseFactory)->createResponse($ex, $request)
);
```

<h1 id="logging">Logging</h1>

Unless you specify otherwise, a <a href="https://github.com/Seldaek/monolog" target="_blank">Monolog</a> logger to log all exceptions to the PHP error log.  However, you can override this with any PSR-3 logger:

```php
use Aphiria\Api\Exceptions\ExceptionResponseFactory;
use Aphiria\Net\Http\ContentNegotiation\ContentNegotiator;
use Aphiria\Net\Http\ContentNegotiation\MediaTypeFormatters\JsonMediaTypeFormatter;
use Aphiria\Net\Http\ContentNegotiation\NegotiatedResponseFactory;
use Monolog\Handler\ErrorLogHandler;
use Monolog\Logger;

// First, set up the factory that will create exception responses
$exceptionResponseFactory = new ExceptionResponseFactory(
    new NegotiatedResponseFactory(
        new ContentNegotiator([
            new JsonMediaTypeFormatter()
        ])
    )
);

// Next, set up our logger
$logger = new Logger('app');
$logger->pushHandler(new SyslogHandler());

// Now, set up our exception handler
$exceptionHandler = new ExceptionHandler($exceptionResponseFactory, $logger);
$exceptionHandler->registerWithPhp();
```

It's possible to specify some rules around the <a href="https://www.php-fig.org/psr/psr-3/#5-psrlogloglevel" target="_blank">PSR-3 log level</a> that an exception returns.  This could be useful for things like logging 500s as critical, but everything else as warnings.  Let's look at an example:

```php
use Aphiria\Api\Exceptions\ExceptionLogLevelFactoryRegistry;

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
$exceptionHandler = new ExceptionHandler(
    $exceptionResponseFactory,
    null, 
    $exceptionLogLevelFactories
);
$exceptionHandler->registerWithPhp();
```

> **Note:** You can register many factories at once using `ExceptionLogLevelFactoryRegistry::registerManyFactories()`.

Passing in an array of PSR-3 log levels will cause only those levels to be logged:

```php
$exceptionHandler = new ExceptionHandler(
    $exceptionResponseFactory,
    null,
    null,
    [LogLevel::CRITICAL, LogLevel::EMERGENCY]
);
```

By default, `LogLevel::ERROR`, `LogLevel::CRITICAL`, `LogLevel::ALERT`, and `LogLevel::EMERGENCY` will be logged if `null` is specified.

You can also control the level of PHP errors that are logged by specifying a bitwise value similar to what's in your _php.ini_:

```php
$exceptionHandler = new ExceptionHandler(
    $exceptionResponseFactory, 
    null, 
    null,
    null,
    E_ALL & ~E_NOTICE
);
$exceptionHandler->registerWithPhp();
```
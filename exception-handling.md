<h1 id="doc-title">Exception Handling</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Global Exception Handler](#global-exception-handler)
2. [Problem Details Exception Renderer](#problem-details-exception-renderer)
   1. [Custom Problem Details Mappings](#custom-problem-details-mappings)
3. [Console Exception Renderer](#console-exception-renderer)
   1. [Output Writers](#output-writers)
3. [Logging](#logging)
   1. [Exception Log Levels](#exception-log-levels)

</div>

</nav>

<h2 id="global-exception-handler">Global Exception Handler</h2>

At some point, your application is going to throw an unhandled exception or shut down unexpectedly.  When this happens, it would be nice to log details about the error and present a nicely-formatted response for the user.  Aphiria provides `GlobalExceptionHandler` to do just this.  It can be used to render exceptions for both HTTP and console applications, and is framework-agnostic.

Let's look at an example:

```php
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Api\Exceptions\ProblemDetailsExceptionRenderer;

$exceptionRenderer = new ProblemDetailsExceptionRenderer();
$globalExceptionHandler = new GlobalExceptionHandler($exceptionRenderer);
// This registers the handler as the default handler in PHP
$globalExceptionHandler->registerWithPhp();
```

That's it.  Now, whenever an unhandled error or exception is thrown, the global exception handler will catch it, [log it](#logging), and [render it](#problem-details-exception-renderer).  We'll go into more details on how to customize it below.

<h2 id="problem-details-exception-renderer">Problem Details Exception Renderer</h2>

`ProblemDetailsExceptionRenderer` is provided out of the box to simplify rendering <a href="https://tools.ietf.org/html/rfc7807" target="_blank">problem details</a> API responses for Aphiria applications.  This renderer tries to create a response using the following steps:
  
1. If a [custom mapping](#custom-problem-details-mappings) exists for the thrown exception, it's used to create a problem details response
2. If no mapping exists, a default 500 problem details response will be returned

By default, when the <a href="https://tools.ietf.org/html/rfc7807#section-3.1" target="_blank">type</a> field is `null`, it is automatically populated with a link to the <a href="https://tools.ietf.org/html/rfc7231#section-6" target="_blank">HTTP status code</a> contained in the problem details.  You can override this behavior by extending `ProblemDetailsExceptionRenderer` and implementing your own `getTypeFromException()`.  Similarly, the exception message is used as the title of the problem details.  If you'd like to customize that, implement your own `ProblemDetailsExceptionRenderer::getTitleFromException()`.

<h3 id="custom-problem-details-mappings">Custom Problem Details Mappings</h3>

You might not want all exceptions to result in a 500.  For example, if you have a `UserNotFoundException`, you might want to map that to a 404.  Here's how:

```php
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Api\Exceptions\ProblemDetailsExceptionRenderer;
use Aphiria\Net\Http\HttpStatusCode;

$exceptionRenderer = new ProblemDetailsExceptionRenderer();
$exceptionRenderer->mapExceptionToProblemDetails(UserNotFoundException::class, status: HttpStatusCode::NotFound);
$globalExceptionHandler = new GlobalExceptionHandler($exceptionRenderer);
$globalExceptionHandler->registerWithPhp();
```

You can also specify other properties in the problem details:

```php
$exceptionRenderer->mapExceptionToProblemDetails(
    OverdrawnException::class,
    type: 'https://example.com/errors/overdrawn',
    title: 'This account is overdrawn',
    detail: fn ($ex) => "Account {$ex->accountId} is overdrawn by {$ex->overdrawnAmount}",
    status: HttpStatusCode::BadRequest,
    instance: fn ($ex) => "https://example.com/accounts/{$ex->accountId}/errors/{$ex->id}",
    extensions: fn ($ex) => ['overdrawnAmount' => $ex->overdrawnAmount]
);
```

> **Note:** All parameters that accept closures with the thrown exception can also take hard-coded values.

When a `ProblemDetails` instance is serialized in a response, all of its extensions are serialized as top-level properties - not as key-value pairs under an `extensions` property.

<h2 id="console-exception-renderer">Console Exception Renderer</h2>

`ConsoleExceptionRenderer` renders exceptions for Aphiria console applications.  To render the exception, it goes through the following steps:

1. If an [output writer](#output-writers) is registered for the thrown exception, it's used
2. Otherwise, the exception message and stack trace is output to the console

<h3 id="output-writers">Output Writers</h3>

Output writers allow you to write errors to the output and return a status code.

```php
use Aphiria\Console\Output\IOutput;
use Aphiria\Console\StatusCodes;
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Console\Exceptions\ConsoleExceptionRenderer;

$exceptionRenderer = new ConsoleExceptionRenderer();
$exceptionRenderer->registerOutputWriter(
    DatabaseNotFound::class,
    function (DatabaseNotFound $ex, IOutput $output) {
        $output->writeln('<fatal>Contact a sysadmin</fatal>');

        return StatusCodes::FATAL;
    }
);

// You can also register many exceptions-to-output writers
$exceptionRenderer->registerManyOutputWriters([
    DatabaseNotFound::class => function (DatabaseNotFound $ex, IOutput $output) {
        $output->writeln('<fatal>Contact a sysadmin</fatal>');

        return StatusCodes::FATAL;
    },
    // ...
]);

$globalExceptionHandler = new GlobalExceptionHandler($exceptionRenderer);
$globalExceptionHandler->registerWithPhp();
```

<h2 id="logging">Logging</h2>

By default, the global exception handler is compatible with any PSR-3 logger, such as <a href="https://github.com/Seldaek/monolog" target="_blank">Monolog</a>.  To use a specific logger, just pass it into the handler:

```php
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Api\Exceptions\ApiExceptionRenderer;
use Monolog\Handler\StreamHandler;
use Monolog\Logger;
use Psr\Log\LogLevel;

$exceptionRenderer = new ApiExceptionRenderer();
$logger = new Logger('app');
$logger->pushHandler(new StreamHandler('/etc/logs/errors.txt', LogLevel::DEBUG));
$globalExceptionHandler = new GlobalExceptionHandler($exceptionRenderer, $logger);
$globalExceptionHandler->registerWithPhp();
```

<h3 id="exception-log-levels">Exception Log Levels</h3>

It's possible to map certain exceptions to a PSR-3 log level.  For example, if you have an exception that means your infrastructure might be down, you can cause it to log as an emergency.

```php
use Aphiria\Exceptions\GlobalExceptionHandler;
use Aphiria\Framework\Api\Exceptions\ApiExceptionRenderer;
use Psr\Log\LogLevel;

$globalExceptionHandler = new GlobalExceptionHandler(new ApiExceptionRenderer());
$globalExceptionHandler->registerLogLevelFactory(
    DatabaseNotFoundException::class,
    fn (DatabaseNotFoundException $ex) => LogLevel::EMERGENCY
);

// You can also register multiple exceptions-to-log-level factories
$globalExceptionHandler->registerManyLogLevelFactories([
    DatabaseNotFoundException::class => fn (DatabaseNotFoundException $ex) => LogLevel::EMERGENCY,
    // ...
]);
```

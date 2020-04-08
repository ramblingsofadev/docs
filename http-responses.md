<h1 id="doc-title">HTTP Responses</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
2. [JSON Responses](#json-responses)
3. [Redirect Responses](#redirect-responses)
4. [Setting Cookies](#setting-response-cookies)
   1. [Deleting Cookies](#deleting-response-cookies)
5. [Writing Responses](#writing-responses)
6. [Serializing Responses](#serializing-responses)

</div>

</nav>

<h2 id="basics">Basics</h2>

Responses are HTTP messages that are sent by servers back to the client.  They contain data like a status code, headers, and body.

```php
use Aphiria\Net\Http\HttpHeaders
use Aphiria\Net\Http\Response;
use Aphiria\Net\Http\StringBody;

$response = new Response(
    200,
    new HttpHeaders(),
    new StringBody('foo')
);

// Get the status code
$response->getStatusCode();

// Set the status code
$response->setStatusCode(500);

// Get the reason phrase
$response->getReasonPhrase(); // "OK"

// Get the protocol version
$response->getProtocolVersion(); // 1.1

// Get the headers (you can add new headers to the returned hash table)
$response->getHeaders();

// Set a header
$response->getHeaders()->add('Foo', 'bar');

// Get the body (could be null)
$response->getBody();

// Set the body
$response->setBody(new StringBody('foo'));
```

<h2 id="json-responses">JSON Responses</h2>

Aphiria provides an easy way to create common responses.  For example, to create a JSON response, use `ResponseFormatter`:

```php
use Aphiria\Net\Http\Formatting\ResponseFormatter;
use Aphiria\Net\Http\Response;

$response = new Response();
(new ResponseFormatter)->writeJson($response, ['foo' => 'bar']);
```

This will set the contents of the response, as well as the appropriate `Content-Type` headers.

<h2 id="redirect-responses">Redirect Responses</h2>

You can also create a redirect response:

```php
use Aphiria\Net\Http\Formatting\ResponseFormatter;
use Aphiria\Net\Http\Response;

$response = new Response();
(new ResponseFormatter)->redirectToUri($response, 'http://example.com');
```

<h2 id="setting-response-cookies">Setting Cookies</h2>

Cookies are headers that are automatically appended to each request from the client to the server.  To set one, use `ResponseFormatter`:

```php
use Aphiria\Net\Http\Formatting\ResponseFormatter;
use Aphiria\Net\Http\Headers\Cookie;

(new ResponseFormatter)->setCookie(
    $response,
    new Cookie(
        'token', // The name
        'abc123', // The value
        time() + 3600, // The expiration (defaults to null, can also be a DateTime)
        '/', // The path (defaults to null)
        'example.com', // The domain (defaults to null)
        true, // Whether or not the cookie is HTTPS-only (defaults to false)
        true, // Whether or not the cookie is not readable by client scripts (defaults to true)
        Cookie::SAME_SITE_LAX // The same-site setting (defaults to lax)
    )
);
```

Use `ResponseFormatter::setCookies()` to set multiple cookies at once.

<h3 id="deleting-response-cookies">Deleting Cookies</h3>

To delete a cookie on the client, call

```php
(new ResponseFormatter)->deleteCookie($response, 'userid');
```

<h2 id="writing-responses">Writing Responses</h2>

Once you're ready to start sending the response back to the client, you can use `ResponseWriter`:

```php
use Aphiria\Net\Http\ResponseWriter;

(new ResponseWriter)->writeResponse($response);
```

By default, this will write the response to the `php://output` stream.  You can override the stream it writes to via the constructor:

```php
$outputStream = new Stream(fopen('path/to/output', 'wb'));
(new ResponseWriter($outputStream))->writeResponse($response);
```

<h2 id="serializing-responses">Serializing Responses</h2>

Aphiria can serialize responses per <a href="https://tools.ietf.org/html/rfc7230#section-3" target="_blank">RFC 7230</a>:

```php
echo (string)$response;
```

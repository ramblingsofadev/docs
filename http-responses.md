# HTTP Responses

## Table of Contents
1. [Basics](#basics)
2. [Creating Responses](#creating-responses)
   1. [Response Headers](#response-headers)
   2. [Response Bodies](#response-bodies)
4. [JSON Responses](#json-responses)
5. [Redirect Responses](#redirect-responses)
6. [Setting Cookies](#setting-response-cookies)
   1. [Deleting Cookies](#deleting-response-cookies)
7. [Writing Responses](#writing-responses)
8. [Serializing Responses](#serializing-responses)

<h2 id="basics">Basics</h2>

Responses are HTTP messages that are sent by servers back to the client.  They contain the following methods:

* `getBody(): ?IHttpBody`
* `getHeaders(): HttpHeaders`
* `getProtocolVersion(): string`
* `getReasonPhrase(): ?string`
* `getStatusCode(): int`
* `setBody(IHttpBody $body): void`
* `setStatusCode(int $statusCode, ?string $reasonPhrase = null): void`

<h2 id="creating-responses">Creating Responses</h2>

Creating a response is easy:

```php
use Aphiria\Net\Http\Response;

$response = new Response();
```

This will create a 200 OK response.  If you'd like to set a different status code, you can either pass it in the constructor or via `Response::setStatusCode()`:

```php
$response = new Response(404);
// Or...
$response->setStatusCode(404);
```

<h3 id="response-headers">Response Headers</h3>

You can set response [headers](#http-headers) via `Response::getHeaders()`:

```php
$response->getHeaders()->add('Content-Type', 'application/json');
```

<h3 id="response-bodies">Response Bodies</h3>

You can pass the [body](#http-bodies) via the response constructor or via `Response::setBody()`:

```php
$response = new Response(200, null, new StringBody('foo'));
// Or...
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
use Aphiria\Net\Http\Cookie;
use Aphiria\Net\Http\Formatting\ResponseFormatter;

(new ResponseFormatter)->setCookie(
    $response,
    new Cookie('userid', 123, 3600)
);
```

`Cookie` accepts the following parameters:

```php
public function __construct(
    string $name,
    $value,
    $expiration = null, // Either Unix timestamp or DateTime to expire
    ?string $path = null,
    ?string $domain = null,
    bool $isSecure = false,
    bool $isHttpOnly = true,
    ?string $sameSite = null
)
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
$outputStream = new Stream(fopen('path/to/output', 'w'));
(new ResponseWriter($outputStream))->writeResponse($response);
```

<h2 id="serializing-responses">Serializing Responses</h2>

Aphiria can serialize responses per <a href="https://tools.ietf.org/html/rfc7230#section-3" target="_blank">RFC 7230</a>:

```php
echo (string)$response;
```

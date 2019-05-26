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

<h1 id="basics">Basics</h1>

Responses are HTTP messages that are sent by servers back to the client.  They contain the following methods:

* `getBody(): ?IHttpBody`
* `getHeaders(): HttpHeaders`
* `getProtocolVersion(): string`
* `getReasonPhrase(): ?string`
* `getStatusCode(): int`
* `setBody(IHttpBody $body): void`
* `setStatusCode(int $statusCode, ?string $reasonPhrase = null): void`

<h1 id="creating-responses">Creating Responses</h1>

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

<h2 id="response-headers">Response Headers</h2>

You can set response [headers](#http-headers) via `Response::getHeaders()`:

```php
$response->getHeaders()->add('Content-Type', 'application/json');
```

<h2 id="response-bodies">Response Bodies</h2>

You can pass the [body](#http-bodies) via the response constructor or via `Response::setBody()`:

```php
$response = new Response(200, null, new StringBody('foo'));
// Or...
$response->setBody(new StringBody('foo'));
```

<h1 id="json-responses">JSON Responses</h1>

Aphiria provides an easy way to create common responses.  For example, to create a JSON response, use `ResponseFormatter`:

```php
use Aphiria\Net\Http\Formatting\ResponseFormatter;
use Aphiria\Net\Http\Response;

$response = new Response();
(new ResponseFormatter)->writeJson($response, ['foo' => 'bar']);
```

This will set the contents of the response, as well as the appropriate `Content-Type` headers.

<h1 id="redirect-responses">Redirect Responses</h1>

You can also create a redirect response:

```php
use Aphiria\Net\Http\Formatting\ResponseFormatter;
use Aphiria\Net\Http\Response;

$response = new Response();
(new ResponseFormatter)->redirectToUri($response, 'http://example.com');
```

<h1 id="setting-response-cookies">Setting Cookies</h1>

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

<h2 id="deleting-response-cookies">Deleting Cookies</h2>

To delete a cookie on the client, call

```php
(new ResponseFormatter)->deleteCookie($response, 'userid');
```

<h1 id="writing-responses">Writing Responses</h1>

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

<h1 id="serializing-responses">Serializing Responses</h1>

Aphiria can serialize responses per <a href="https://tools.ietf.org/html/rfc7230#section-3" target="_blank">RFC 7230</a>:

```php
echo (string)$response;
```

<h1 id="http-headers">HTTP Headers</h1>

Headers provide metadata about the HTTP message.  In Aphiria, they're implemented by `Aphiria\Net\Http\HttpHeaders`, which extends  [`Opulence\Collections\HashTable`](https://www.opulencephp.com/docs/1.1/collections#hash-tables).  On top of the methods provided by `HashTable`, they also provide the following methods:

* `getFirst(string $name): mixed`
* `tryGetFirst(string $name, &$value): bool`

> **Note:** Header names that are passed into the methods in `HttpHeaders` are automatically normalized to Train-Case.  In other words, `foo_bar` will become `Foo-Bar`.

<h1 id="http-bodies">HTTP Bodies</h1>

HTTP bodies contain data associated with the HTTP message, and are optional.  They're represented by `Aphiria\Net\Http\IHttpBody`.  They provide a few methods to read and write their contents to streams and to strings:

* `__toString(): string`
* `readAsStream(): IStream`
* `readAsString(): string`
* `writeToStream(IStream $stream): void`

<h1 id="string-bodies">String Bodies</h1>

HTTP bodies are most commonly represented as strings.  Aphiria makes it easy to create a string body via `StringBody`:

```php
use Aphiria\Net\Http\StringBody;

$body = new StringBody('foo');
```

<h1 id="stream-bodies">Stream Bodies</h1>

Sometimes, bodies might be too big to hold entirely in memory.  This is where `StreamBody` comes in handy:

```php
use Aphiria\IO\Streams\Stream;
use Aphiria\Net\Http\StreamBody;

$stream = new Stream(fopen('foo.txt', 'r+'));
$body = new StreamBody($stream);
```

<h1 id="uris">URIs</h1>

A URI identifies a resource, typically over a network.  They contain such information as the scheme, host, port, path, query string, and fragment.  Aphiria represents them in `Aphiria\Net\Uri`, and they include the following methods:

* `__toString(): string`
* `getAuthority(bool $includeUserInfo = true): ?string`
* `getFragment(): ?string`
* `getHost(): ?string`
* `getPassword(): ?string`
* `getPath(): ?string`
* `getPort(): ?int`
* `getQueryString(): ?string`
* `getScheme(): ?string`
* `getUser(): ?string`

To create an instance of `Uri`, pass the raw URI string into the constructor:

```php
use Aphiria\Net\Uri;

$uri = new Uri('https://example.com/foo?bar=baz#blah');
```
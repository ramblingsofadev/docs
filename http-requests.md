<h1 id="doc-title">HTTP Requests</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
2. [Creating Requests](#creating-requests)
   1. [Creating Requests From Superglobals](#creating-requests-from-superglobals)
   2. [Request Builders](#request-builders)
3. [Headers](#headers)
4. [Bodies](#bodies)
   1. [String Bodies](#string-bodies)
   2. [Stream Bodies](#stream-bodies)
5. [URIs](#uris)
6. [Getting POST Data](#getting-post-data)
7. [Getting Query String Data](#getting-query-string-data)
8. [JSON Requests](#json-requests)
9. [Getting Cookies](#getting-request-cookies)
10. [Getting Client IP Address](#getting-client-ip-address)
11. [Header Parameters](#header-parameters)
12. [Serializing Requests](#serializing-requests)
13. [Multipart Requests](#multipart-requests)
    1. [Saving Uploaded Files](#saving-uploaded-files)
    2. [Getting the MIME Type of the Body](#getting-mime-type-of-body)
    3. [Creating Multipart Requests](#creating-multipart-requests)
14. [Trusted Proxies](#trusted-proxies)

</div>

</nav>

<h2 id="basics">Basics</h2>

Requests are HTTP messages sent by clients to servers.  They contain data like the URI that was requested, the HTTP method (eg `GET`), the headers, and the body (if one was present).

Let's create a request:

```php
use Aphiria\Net\Http\Request;
use Aphiria\Net\Http\StringBody;
use Aphiria\Net\Uri;

$request = new Request('GET', new Uri('https://example.com'));

// Grab the HTTP method, eg "GET"
$method = $request->method;

// Grab the URI:
$uri = $request->uri;

// Grab the protocol version, eg 1.1:
$protocolVersion = $request->protocolVersion;

// Grab the headers (you can add new headers to the returned hash table)
$headers = $request->headers;

// Set a header
$request->headers->add('Foo', 'bar');

// Get the body (could be null)
$body = $request->body;

// Set the body
$request->body = new StringBody('foo');

// Grab any special metadata properties (a custom hash table that you can define)
$properties = $request->properties;
```

<h2 id="creating-requests">Creating Requests</h2>

Manually creating a request is easy:

```php
use Aphiria\Net\Http\Headers;
use Aphiria\Net\Http\Request;
use Aphiria\Net\Http\StringBody;
use Aphiria\Net\Uri;

$request = new Request(
    'GET',
    new Uri('https://example.com'),
    new Headers(),
    new StringBody('foo')
);
```

<h3 id="creating-requests-from-superglobals">Creating Requests From Superglobals</h3>

PHP has superglobal arrays that store information about the requests.  They're a mess, architecturally-speaking.  Aphiria attempts to insulate developers from the nastiness of superglobals by giving you a simple method to create requests and responses.  To create a request, use `RequestFactory`:

```php
use Aphiria\Net\Http\RequestFactory;

$request = new RequestFactory()->createRequestFromSuperglobals($_SERVER);
```

Aphiria reads all the information it needs from the `$_SERVER` superglobal - it doesn't need the others.

<h3 id="request-builders">Request Builders</h3>

Aphiria comes with a fluent syntax for building your requests, which is somewhat similar to PSR-7.  Let's look at a simple example:

```php
use Aphiria\Net\Http\RequestBuilder;

$request = new RequestBuilder()->withMethod('GET')
    ->withUri('http://example.com')
    ->withHeader('Cookie', 'foo=bar')
    ->build();
```

You can specify a body of a request:

```php
use Aphiria\Net\Http\StringBody;

$request = new RequestBuilder()->withMethod('POST')
    ->withUri('http://example.com/users')
    ->withBody(new StringBody('{"name":"Dave"}'))
    ->withHeader('Content-Type', 'application/json')
    ->build();
```

> **Note:** If you specify a body, you **must** also specify a content type.  You can also pass in `null` to remove a body from the request.

You can specify multiple headers in one call:

```php
$request = new RequestBuilder()->withManyHeaders(['Foo' => 'bar', 'Baz' => 'buzz'])
    ->build();
```

You can also set any request properties:

```php
$request = new RequestBuilder()->withProperty('routeVars', ['id' => 123])
    ->build();
```

If you'd like to use a different request target type besides origin form, you may:

```php
use Aphiria\Net\Http\RequestTargetType;

$request = new RequestBuilder()->withRequestTargetType(RequestTargetType::AbsoluteForm)
    ->build();
```

> **Note:**  Keep in mind that each `with*()` method will return a clone of the request builder.

Aphiria also has a [negotiated](content-negotiation.md) request builder that can serialize your models to a request body.

```php
use Aphiria\ContentNegotiation\NegotiatedRequestBuilder;

$request = new NegotiatedRequestBuilder()->withMethod('POST')
    ->withUri('http://example.com/users')
    ->withBody(new User('Dave'))
    ->build();
```

> **Note:** By default, negotiated request bodies will be serialized to JSON, but you can specify a different default content type, eg `new NegotiatedRequestBuilder(defaultContentType: 'text/xml')`.

<h2 id="headers">Headers</h2>

Headers provide metadata about the HTTP message.  In Aphiria, they are an extension of [`HashTable`](collections.md#hash-tables), and also provide the following methods:

* `getFirst(string $name): mixed`
* `tryGetFirst(string $name, mixed &$value): bool`

> **Note:** Header names that are passed into the methods in `Headers` are automatically normalized to Train-Case.  In other words, `foo_bar` will become `Foo-Bar`.

<h2 id="bodies">Bodies</h2>

HTTP bodies contain data associated with the HTTP message, and are optional.  They're represented by `Aphiria\Net\Http\IBody`, and provide a few methods to read and write their contents to streams and to strings:

```php
use Aphiria\IO\Streams\Stream;
use Aphiria\Net\Http\StringBody;

$body = new StringBody('foo');

// Read the body as a stream
$stream = $body->readAsStream();

// Read the body as a string
$body->readAsString(); // "foo"
(string)$body; // "foo"

// Write the body to a stream
$streamToWriteTo = new Stream(fopen('php://temp', 'w+b'));
$body->writeToStream($streamToWriteTo);
```

<h3 id="string-bodies">String Bodies</h3>

HTTP bodies are most commonly represented as strings.

```php
use Aphiria\Net\Http\StringBody;

$body = new StringBody('foo');
```

<h3 id="stream-bodies">Stream Bodies</h3>

Sometimes, bodies might be too big to hold entirely in memory.  This is where `StreamBody` comes in handy.

```php
use Aphiria\IO\Streams\Stream;
use Aphiria\Net\Http\StreamBody;

$stream = new Stream(fopen('foo.txt', 'r+b'));
$body = new StreamBody($stream);
```

<h2 id="uris">URIs</h2>

A URI identifies a resource, typically over a network.  They contain such information as the scheme, host, port, path, query string, and fragment.  Aphiria represents them in `Aphiria\Net\Uri`, and they include the following methods:

```php
use Aphiria\Net\Uri;

$uri = new Uri('https://dave:abc123@example.com:443/foo?bar=baz#hash');
$uri->scheme; // "https"
$uri->user; // "dave"
$uri->password; // "abc123"
$uri->host; // "example.com"
$uri->port; // 443
$uri->path; // "/foo"
$uri->queryString; // "bar=baz"
$uri->fragment; // "hash"
$uri->getAuthority(); // "//dave:abc123@example.com:443"
```

To serialize a URI, just cast it to a string:

```php
(string)$uri; // "https://dave:abc123@example.com:443/foo?bar=baz#hash"
```

<h2 id="getting-post-data">Getting POST Data</h2>

In vanilla PHP, you can read URL-encoded form data via the `$_POST` superglobal.  Aphiria gives you a helper to parse the body of form requests into a [hash table](collections.md#hash-tables).

```php
use Aphiria\Net\Http\Formatting\RequestParser;

// Let's assume the raw body is "email=foo%40bar.com"
$formInput = new RequestParser()->readAsFormInput($request);
echo $formInput->get('email'); // "foo@bar.com"
```

<h2 id="getting-query-string-data">Getting Query String Data</h2>

In vanilla PHP, query string data is read from the `$_GET` superglobal.  In Aphiria, it's stored in the request's URI.  `Uri::queryString` returns the raw query string - to return it as an [immutable hash table](collections.md#immutable-hash-tables), use `RequestParser`:

```php
use Aphiria\Net\Http\Formatting\RequestParser;

// Assume the query string was "?foo=bar"
$queryStringParams = new RequestParser()->parseQueryString($request);
echo $queryStringParams->get('foo'); // "bar"
```

<h2 id="json-requests">JSON Requests</h2>

To check if a request is a JSON request, call

```php
use Aphiria\Net\Http\Formatting\RequestParser;

$isJson = new RequestParser()->isJson($request);
```

Rather than having to parse a JSON body yourself, you can use `RequestParser` to do it for you:

```php
use Aphiria\Net\Http\Formatting\RequestParser;

$json = new RequestParser()->readAsJson($request);
```

<h2 id="getting-request-cookies">Getting Cookies</h2>

Aphiria has a helper to grab cookies from request headers as an [immutable hash table](collections.md#immutable-hash-tables):

```php
use Aphiria\Net\Http\Formatting\RequestParser;

// Assume the request contained the header "Cookie: userid=123"
$cookies = new RequestParser()->parseCookies($request);
echo $cookies->get('userid'); // "123"
```

<h2 id="getting-client-ip-address">Getting Client IP Address</h2>

If you use the [`RequestFactory`](#creating-request-from-superglobals) to create your request, the client IP address will be added to the request property `CLIENT_IP_ADDRESS`.  To make it easier to grab this value, you can use `RequestParser` to retrieve it:

```php
use Aphiria\Net\Http\Formatting\RequestParser;

$clientIPAddress = new RequestParser()->getClientIPAddress($request);
```

> **Note:** This will take into consideration any [trusted proxy header values](#trusted-proxies) when determining the original client IP address.

<h2 id="header-parameters">Header Parameters</h2>

Some header values are semicolon delimited, eg `Content-Type: text/html; charset=utf-8`.  Aphiria provides an easy way to grab those key-value pairs:

```php
$contentTypeValues = $requestParser->parseParameters($request, 'Content-Type');
// Keys without values will return null:
echo $contentTypeValues->get('text/html'); // null
echo $contentTypeValues->get('charset'); // "utf-8"
```

<h2 id="serializing-requests">Serializing Requests</h2>

You can serialize a request per <a href="https://tools.ietf.org/html/rfc7230#section-3" target="_blank">RFC 7230</a> by casting it to a string:

```php
echo (string)$request;
```

By default, this will use <a href="https://tools.ietf.org/html/rfc7230#section-5.3.1" target="_blank">origin-form</a> for the request target, but you can override the request type via the constructor:

```php
use Aphiria\Net\Http\RequestTargetType;

$request = new Request(
    'GET',
    new Uri('https://example.com/foo?bar'),
    requestTargetType: RequestTargetType::AuthorityForm
);
```

The following request target types may be used:

* <a href="https://tools.ietf.org/html/rfc7230#section-5.3.2" target="_blank">`RequestTargetType::AbsoluteForm`</a>
* <a href="https://tools.ietf.org/html/rfc7230#section-5.3.4" target="_blank">`RequestTargetType::AsteriskForm`</a>
* <a href="https://tools.ietf.org/html/rfc7230#section-5.3.3" target="_blank">`RequestTargetType::AuthorityForm`</a>
* <a href="https://tools.ietf.org/html/rfc7230#section-5.3.1" target="_blank">`RequestTargetType::OriginForm`</a>

<h2 id="multipart-requests">Multipart Requests</h2>

Multipart requests contain multiple bodies, each with headers.  That's actually how file uploads work - each file gets a body with headers indicating the name, type, and size of the file.  Aphiria can parse these multipart bodies into a `MultipartBody`, which extends [`StreamBody`](#stream-bodies).  It contains additional properties to get the boundary and the list of `MultipartBodyPart` objects that make up the body:

```php
$multipartBody->boundary; // string
$multipartBody->parts; // MultipartBodyPart[]
```

You can check if a request is a multipart request:

```php
use Aphiria\Net\Http\Formatting\RequestParser;

$isMultipart = new RequestParser()->isMultipart($request);
```

To parse a request body as a multipart body, call

```php
use Aphiria\Net\Http\Formatting\RequestParser;

$multipartBody = new RequestParser()->readAsMultipart($request);
```

Each `MultipartBodyPart` contains the following properties:

```php
$multipartBodyPart->body; // ?IBody
$multipartBodyPart->headers; // Headers
```

<h3 id="saving-uploaded-files">Saving Uploaded Files</h3>

To save a multipart body's parts to files in a memory-efficient manner, read each part as a stream and copy it to the destination path:

```php
foreach ($multipartBody->parts as $multipartBodyPart) {
    $bodyStream = $multipartBodyPart->body->readAsStream();
    $bodyStream->rewind();
    $bodyStream->copyToStream(new Stream(fopen('path/to/copy/to/' . uniqid(), 'wb')));
}
```

<h3 id="getting-mime-type-of-body">Getting the MIME Type of the Body</h3>

To grab the actual MIME type of an HTTP body, call

```php
$actualMimeType = new RequestParser()->getActualMimeType($multipartBodyPart);
```

To get the MIME type that was specified by the client, call

```php
$clientMimeType = new RequestParser()->getClientMimeType($multipartBodyPart);
```

<h3 id="creating-multipart-requests">Creating Multipart Requests</h3>

The Net library makes it straightforward to create a multipart request manually.  The following example creates a request to upload two images:

```php
use Aphiria\Net\Http\Headers;
use Aphiria\Net\Http\MultipartBody;
use Aphiria\Net\Http\MultipartBodyPart;
use Aphiria\Net\Http\Request;
use Aphiria\Net\Http\StreamBody;
use Aphiria\Net\Uri;

// Build the first image's headers and body
$image1Headers = new Headers();
$image1Headers->add('Content-Disposition', 'form-data; name="image1"; filename="foo.png"');
$image1Headers->add('Content-Type', 'image/png');
$image1Body = new StreamBody(fopen('path/to/foo.png', 'rb'));

// Build the second image's headers and body
$image2Headers = new Headers();
$image2Headers->add('Content-Disposition', 'form-data; name="image2"; filename="bar.png"');
$image2Headers->add('Content-Type', 'image/png');
$image2Body = new StreamBody(fopen('path/to/bar.png', 'rb'));

// Build the request's headers and body
$body = new MultipartBody([
    new MultipartBodyPart($image1Headers, $image1Body),
    new MultipartBodyPart($image2Headers, $image2Body)
]);
$headers = new Headers();
$headers->add('Content-Type', "multipart/form-data; boundary={$body->boundary}");

// Build the request
$request = new Request(
    'POST',
    new Uri('https://example.com'),
    $headers,
    $body
);
```

<h2 id="trusted-proxies">Trusted Proxies</h2>

If you're using a load balancer or some sort of proxy server, you'll need to add it to the list of trusted proxies.  You can also use your proxy to set custom, trusted headers.  You may specify them in the factory constructor:

```php
// The client IP will be read from the "X-My-Proxy-Ip" header when using a trusted proxy
$factory = new RequestFactory(['192.168.1.1'], ['HTTP_CLIENT_IP' => 'X-My-Proxy-Ip']);
$request = $factory->createRequestFromSuperglobals($_SERVER);
```

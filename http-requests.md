<h1 id="doc-title">HTTP Requests</h1>

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
2. [Creating Requests](#creating-requests)
   1. [Creating Requests From Superglobals](#creating-requests-from-superglobals)
   2. [Trusted Proxies](#trusted-proxies)
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

<h2 id="basics">Basics</h2>

Requests are HTTP messages sent by clients to servers.  They contain the following methods:

* `getBody(): ?IHttpBody`
* `getHeaders(): HttpHeaders`
* `getMethod(): string`
* `getProperties(): IDictionary`
* `getProtocolVersion(): string`
* `getUri(): Uri`
* `setBody(IHttpBody $body): void`

> **Note:** The properties dictionary is a useful place to store metadata about a request, eg route variables.

<h2 id="creating-requests">Creating Requests</h2>

Creating a request is easy:

```php
use Aphiria\Net\Http\Request;
use Aphiria\Net\Uri;

$request = new Request('GET', new Uri('https://example.com'));
```

You can set HTTP headers by calling

```php
$request->getHeaders()->add('Foo', 'bar');
```

You can either set the body via the constructor or via `Request::setBody()`:

```php
use Aphiria\Net\Http\StringBody;

// Via constructor:
$body = new StringBody('foo');
$request = new Request('POST', new Uri('https://example.com'), null, $body);

// Or via setBody():
$request->setBody($body);
```

<h3 id="creating-requests-from-superglobals">Creating Requests From Superglobals</h3>

PHP has superglobal arrays that store information about the requests.  They're a mess, architecturally-speaking.  Aphiria attempts to insulate developers from the nastiness of superglobals by giving you a simple method to create requests and responses.  To create a request, use `RequestFactory`:

```php
use Aphiria\Net\Http\RequestFactory;

$request = (new RequestFactory)->createRequestFromSuperglobals($_SERVER);
```

Aphiria reads all the information it needs from the `$_SERVER` superglobal - it doesn't need the others.

<h3 id="trusted-proxies">Trusted Proxies</h3>

If you're using a load balancer or some sort of proxy server, you'll need to add it to the list of trusted proxies.  You can also use your proxy to set custom, trusted headers.  You may specify them in the factory constructor:

```php
// The client IP will be read from the "X-My-Proxy-Ip" header when using a trusted proxy
$factory = new RequestFactory(['192.168.1.1'], ['HTTP_CLIENT_IP' => 'X-My-Proxy-Ip']);
$request = $factory->createRequestFromSuperglobals($_SERVER);
```

<h2 id="headers">Headers</h2>

Headers provide metadata about the HTTP message.  In Aphiria, they're implemented by `Aphiria\Net\Http\HttpHeaders`, which extends  [`Aphiria\Collections\HashTable`](https://www.opulencephp.com/docs/1.1/collections#hash-tables).  On top of the methods provided by `HashTable`, they also provide the following methods:

* `getFirst(string $name): mixed`
* `tryGetFirst(string $name, &$value): bool`

> **Note:** Header names that are passed into the methods in `HttpHeaders` are automatically normalized to Train-Case.  In other words, `foo_bar` will become `Foo-Bar`.

<h2 id="bodies">Bodies</h2>

HTTP bodies contain data associated with the HTTP message, and are optional.  They're represented by `Aphiria\Net\Http\IHttpBody`.  They provide a few methods to read and write their contents to streams and to strings:

* `__toString(): string`
* `readAsStream(): IStream`
* `readAsString(): string`
* `writeToStream(IStream $stream): void`

<h3 id="string-bodies">String Bodies</h3>

HTTP bodies are most commonly represented as strings.  Aphiria makes it easy to create a string body via `StringBody`:

```php
use Aphiria\Net\Http\StringBody;

$body = new StringBody('foo');
```

<h3 id="stream-bodies">Stream Bodies</h3>

Sometimes, bodies might be too big to hold entirely in memory.  This is where `StreamBody` comes in handy:

```php
use Aphiria\IO\Streams\Stream;
use Aphiria\Net\Http\StreamBody;

$stream = new Stream(fopen('foo.txt', 'r+'));
$body = new StreamBody($stream);
```

<h2 id="uris">URIs</h2>

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

<h2 id="getting-post-data">Getting POST Data</h2>

In vanilla PHP, you can read URL-encoded form data via the `$_POST` superglobal.  Aphiria gives you a helper to parse the body of form requests into a [dictionary](https://www.opulencephp.com/docs/1.1/collections#hash-tables).

```php
use Aphiria\Net\Http\Formatting\RequestParser;

// Let's assume the raw body is "email=foo%40bar.com"
$formInput = (new RequestParser)->readAsFormInput($request);
echo $formInput->get('email'); // "foo@bar.com"
```

<h2 id="getting-query-string-data">Getting Query String Data</h2>

In vanilla PHP, query string data is read from the `$_GET` superglobal.  In Aphiria, it's stored in the request's URI.  `Uri::getQueryString()` returns the raw query string - to return it as an [immutable dictionary](https://www.opulencephp.com/docs/1.1/collections#immutable-hash-tables), use `RequestParser`:

```php
use Aphiria\Net\Http\Formatting\RequestParser;

// Assume the query string was "?foo=bar"
$queryStringParams = (new RequestParser)->parseQueryString($request);
echo $queryStringParams->get('foo'); // "bar"
```

<h2 id="json-requests">JSON Requests</h2>

To check if a request is a JSON request, call

```php
use Aphiria\Net\Http\Formatting\RequestParser;

$isJson = (new RequestParser)->isJson($request);
```

Rather than having to parse a JSON body yourself, you can use `RequestParser` to do it for you:

```php
use Aphiria\Net\Http\Formatting\RequestParser;

$json = (new RequestParser)->readAsJson($request);
```

<h2 id="getting-request-cookies">Getting Cookies</h2>

Aphiria has a helper to grab cookies from request headers as an [immutable dictionary](https://www.opulencephp.com/docs/1.1/collections#immutable-hash-tables):

```php
use Aphiria\Net\Http\Formatting\RequestParser;

$cookies = (new RequestParser)->parseCookies($request);
$cookies->get('userid');
```

<h2 id="getting-client-ip-address">Getting Client IP Address</h2>

If you use the [`RequestFactory`](#creating-request-from-superglobals) to create your request, the client IP address will be added to the request property `CLIENT_IP_ADDRESS`.  To make it easier to grab this value, you can use `RequestParser` to retrieve it:

```php
use Aphiria\Net\Http\Formatting\RequestParser;

$clientIPAddress = (new RequestParser)->getClientIPAddress($request);
```

> **Note:** This will take into consideration any [trusted proxy header values](#trusted-proxies) when determining the original client IP address.

<h2 id="header-parameters">Header Parameters</h2>

Some header values are semicolon delimited, eg `Content-Type: text/html; charset=utf-8`.  It's sometimes convenient to grab those key => value pairs:

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
use Aphiria\Net\Http\RequestTargetTypes;

$request = new Request(
    'GET',
    new Uri('https://example.com/foo?bar'),
    null,
    null,
    null,
    '1.1',
    RequestTargetTypes::AUTHORITY_FORM
);
```

The following request target types may be used:

* <a href="https://tools.ietf.org/html/rfc7230#section-5.3.2" target="_blank">`RequestTargetTypes::ABSOLUTE_FORM`</a>
* <a href="https://tools.ietf.org/html/rfc7230#section-5.3.4" target="_blank">`RequestTargetTypes::ASTERISK_FORM`</a>
* <a href="https://tools.ietf.org/html/rfc7230#section-5.3.3" target="_blank">`RequestTargetTypes::AUTHORITY_FORM`</a>
* <a href="https://tools.ietf.org/html/rfc7230#section-5.3.1" target="_blank">`RequestTargetTypes::ORIGIN_FORM`</a>

<h2 id="multipart-requests">Multipart Requests</h2>

Multipart requests contain multiple bodies, each with headers.  That's actually how file uploads work - each file gets a body with headers indicating the name, type, and size of the file.  Aphiria can parse these multipart bodies into a `MultipartBody`, which extends [`StreamBody`](#stream-bodies).  It contains additional methods to get the boundary and the list of `MultipartBodyPart` objects that make up the body:

* `getBoundary(): string`
* `getParts(): MultipartBodyPart[]`

You can check if a request is a multipart request:

```php
use Aphiria\Net\Http\Formatting\RequestParser;

$isMultipart = (new RequestParser)->isMultipart($request);
```

To parse a request body as a multipart body, call

```php
use Aphiria\Net\Http\Formatting\RequestParser;

$multipartBody = (new RequestParser)->readAsMultipart($request);
```

Each `MultipartBodyPart` contains the following methods:

* `getBody(): ?IHttpBody`
* `getHeaders(): HttpHeaders`

<h3 id="saving-uploaded-files">Saving Uploaded Files</h3>

To save a multipart body's parts to files in a memory-efficient manner, read each part as a stream and copy it to the destination path:

```php
foreach ($multipartBody->getParts() as $multipartBodyPart) {
    $bodyStream = $multipartBodyPart->getBody()->readAsStream();
    $bodyStream->rewind();
    $bodyStream->copyToStream(new Stream(fopen('path/to/copy/to/' . uniqid(), 'w')));
}
```

<h3 id="getting-mime-type-of-body">Getting the MIME Type of the Body</h3>

To grab the MIME type of an HTTP body, call

```php
(new RequestParser)->getMimeType($multipartBodyPart);
```

<h3 id="creating-multipart-requests">Creating Multipart Requests</h3>

The Net library makes it straightforward to create a multipart request manually.  The following example creates a request to upload two images:

```php
use Aphiria\Net\Http\HttpHeaders;
use Aphiria\Net\Http\MultipartBody;
use Aphiria\Net\Http\MultipartBodyPart;
use Aphiria\Net\Http\Request;
use Aphiria\Net\Http\StreamBody;
use Aphiria\Net\Uri;

// Build the first image's headers and body
$image1Headers = new HttpHeaders();
$image1Headers->add('Content-Disposition', 'form-data; name="image1"; filename="foo.png"');
$image1Headers->add('Content-Type', 'image/png');
$image1Body = new StreamBody(fopen('path/to/foo.png', 'r'));

// Build the second image's headers and body
$image2Headers = new HttpHeaders();
$image2Headers->add('Content-Disposition', 'form-data; name="image2"; filename="bar.png"');
$image2Headers->add('Content-Type', 'image/png');
$image2Body = new StreamBody(fopen('path/to/bar.png', 'r'));

// Build the request's headers and body
$body = new MultipartBody([
    new MultipartBodyPart($image1Headers, $image1Body),
    new MultipartBodyPart($image2Headers, $image2Body)
]);
$headers = new HttpHeaders();
$headers->add('Content-Type', "multipart/form-data; boundary={$body->getBoundary()}");

// Build the request
$request = new Request(
    'POST',
    new Uri('https://example.com'),
    $headers,
    $body
);
```

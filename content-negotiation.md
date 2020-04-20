<h1 id="doc-title">Content Negotiation</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
2. [Negotiating Requests](#negotiating-requests)
3. [Negotiating Responses](#negotiating-responses)
   1. [Negotiating Language](#negotiating-language)
4. [Media Type Formatters](#media-type-formatters)

</div>

</nav>

<h2 id="basics">Basics</h2>

Content negotiation is a process between the client and server to determine how to best process a request and serve content back to the client.  This negotiation is typically done via headers, where the client says "Here's the type of content I'd prefer (eg JSON, XMl, etc)", and the server trying to accommodate the client's preferences.  For example, the process can involve negotiating the following for requests and responses per the <a href="https://www.w3.org/Protocols/rfc2616/rfc2616-sec12.html" target="_blank">HTTP spec</a>:

* Content type
  * Controlled by the `Content-Type` and `Accept` headers
  * Dictates the [media type formatter](#media-type-formatters) to use
* Character encoding
  * Controlled by the `Content-Type` and `Accept-Charset` headers by default
* Language
  * Controlled by the `Content-Language` and `Accept-Language` headers by default

Setting up your content negotiator is straightforward:

```php
use Aphiria\Net\Http\ContentNegotiation\AcceptCharsetEncodingMatcher;
use Aphiria\Net\Http\ContentNegotiation\AcceptLanguageMatcher;
use Aphiria\Net\Http\ContentNegotiation\ContentNegotiator;
use Aphiria\Net\Http\ContentNegotiation\MediaTypeFormatterMatcher;
use Aphiria\Net\Http\ContentNegotiation\MediaTypeFormatters\FormUrlEncodedMediaTypeFormatter;
use Aphiria\Net\Http\ContentNegotiation\MediaTypeFormatters\JsonMediaTypeFormatter;

// Register whatever media type formatters you support
$mediaTypeFormatters = [
    new JsonMediaTypeFormatter(),
    new FormUrlEncodedMediaTypeFormatter()
];
$contentNegotiator = new ContentNegotiator(
    $mediaTypeFormatters, 
    new MediaTypeFormatterMatcher($mediaTypeFormatters),
    new AcceptCharsetEncodingMatcher(),
    new AcceptLanguageMatcher(['en'])
);
```

Now you're ready to start [negotiating](#negotiating-requests).

> **Note:** `AcceptLanguageMatcher` uses language tags from <a href="https://tools.ietf.org/html/rfc5646" target="_blank">RFC 5646</a>, and follows the lookup rules in <a href="https://tools.ietf.org/html/rfc4647#section-3.4" target="_blank">RFC 4647 Section 3.4</a>.

<h2 id="negotiating-requests">Negotiating Requests</h2>

Let's build off of the [previous example](#basics) and negotiate a request.  Let's assume the raw request looked something like this:

```http
POST https://example.com/users HTTP/1.1
Content-Type: application/json; charset=utf-8
Content-Language: en-US
Encoding: UTF-8
Accept: application/json, text/xml
Accept-Language: en-US, en
Accept-Charset: utf-8, utf-16

{"id":123,"email":"foo@example.com"}
```

Here's how we'd negotiate the request content:

```php
use App\Users\User;

// Assume the request was already instantiated
$result = $contentNegotiator->negotiateRequestContent(User::class, $request);

// The properties of a content negotiation result:
$mediaTypeFormatter = $result->getFormatter(); // An instance of JsonMediaTypeFormatter
$mediaType = $result->getMediaType(); // "application/json"
$encoding = $result->getEncoding(); // "utf-8"
$language = $result->getLanguage(); // "en-us"
```

> **Note:** Any of these properties could be null if they could not be negotiated.

With these results, we know everything we need to try to deserialize the request body:

```php
$user = $mediaTypeFormatter->readFromStream($request->getBody()->readAsStream(), User::class);
echo $user->getId(); // 123
echo $user->getEmail(); // "foo@example.com"
```

<h2 id="negotiating-responses">Negotiating Responses</h2>

We negotiate the response content by inspecting the `Accept`, `Accept-Charset`, and `Accept-Language` headers.  If those headers are missing, we default to using the first media type formatter that can write the response body.

Constructing a response with all the appropriate headers is a little involved when doing it manually, which is why Aphiria provides `NegotiatedResponseFactory` to handle it for you:

```php
use Aphiria\Net\Http\ContentNegotiation\NegotiatedResponseFactory;

$responseFactory = new NegotiatedResponseFactory($contentNegotiator);
// Assume $user is a POPO User object
$response = $responseFactory->createResponse($request, 200, null, $user);
```

Our response will look something like the following:

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Language: en-US
Content-Length: 36

{"id":123,"email":"foo@example.com"}
```

<h3 id="negotiating-language">Negotiating Language</h3>

By default, `ContentNegotiator` uses `AcceptLanguageMatcher` to find the best language to respond in from the `Accept-Language` header.  However, if your locale is, for example, set as a query string parameter, you can use a custom language matcher and inject it into your `ContentNegotiator`.

```php
use Aphiria\Net\Http\ContentNegotiation\ILanguageMatcher;
use Aphiria\Net\Http\Formatting\RequestParser;
use Aphiria\Net\Http\IRequest;

final class QueryStringLanguageMatcher implements ILanguageMatcher
{
    private RequestParser $requestParser;
   
    public function __construct()
    {
        $this->requestParser = new RequestParser();
    }

    public function getBestLanguageMatch(IRequest $request): ?string
    {
        $queryStringVars = $this->requestParser->parseQueryString($request);
        $bestLanguage = null;
        
        if ($queryStringVars->tryGet('locale', $bestLanguage)) {
            return $bestLanguage;
        }

        return null;
    }
}
```

<h2 id="media-type-formatters">Media Type Formatters</h2>

Media type formatters can read and write a particular data format to a stream.  You can get the media type formatter from `ContentNegotiationResult`, and use it to deserialize a request body to a particular type (`User` in this example):

```php
$mediaTypeFormatter = $result->getFormatter();
$mediaTypeFormatter->readFromStream($request->getBody(), User::class);
```

Similarly, you can serialize a value and write it to the response body:

```php
$mediaTypeFormatter->writeToStream($valueToWrite, $response->getBody());
```

Aphiria provides the following formatters out of the box:

* `FormUrlEncodedMediaTypeFormatter`
* `HtmlMediaTypeFormatter`
* `JsonMediaTypeFormatter`
* `PlainTextMediaTypeFormatter`

Under the hood, `FormUrlEncodedMediaTypeFormatter` and `JsonMediaTypeFormatter` use Aphiria's [serialization library](serialization.md) to (de)serialize values.  `HtmlMediaTypeFormatter` and `PlainTextMediaTypeFormatter` only handle strings - they do not deal with objects or arrays.

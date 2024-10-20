<h1 id="doc-title">Content Negotiation</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
2. [Negotiating Requests](#negotiating-requests)
3. [Negotiating Responses](#negotiating-responses)
   1. [Negotiating Language](#negotiating-language)
4. [Media Type Formatters](#media-type-formatters)
   1. [Customizing (De)Serialization](#customizing-deserialization)

</div>

</nav>

<h2 id="basics">Basics</h2>

Content negotiation is a process between the client and server to determine how to best process a request and serve content back to the client.  This negotiation is typically done via headers, where the client says "Here's the type of content I'd prefer (eg JSON, XML, etc)", and the server trying to accommodate the client's preferences.  For example, the process can involve negotiating the following for requests and responses per the <a href="https://www.w3.org/Protocols/rfc2616/rfc2616-sec12.html" target="_blank">HTTP spec</a>:

* Content type
  * Controlled by the `Content-Type` header for requests, and the `Accept` header for responses
  * Dictates the [media type formatter](#media-type-formatters) to use
* Character encoding
  * Controlled by the `Content-Type` header for requests, and the `Accept-Charset` header for responses
* Language
  * Controlled by the `Content-Language` header for requests, and the `Accept-Language` header for responses

Setting up your content negotiator with default settings is trivial:

```php
use Aphiria\ContentNegotiation\ContentNegotiator;

$contentNegotiator = new ContentNegotiator();
```

This will create a negotiator with JSON, XML, HTML, and plain text media type formatters.

If you'd like to customize things like [media type formatters](#media-type-formatters) and supported languages, you can override the defaults:

```php
use Aphiria\ContentNegotiation\AcceptCharsetEncodingMatcher;
use Aphiria\ContentNegotiation\AcceptLanguageMatcher;
use Aphiria\ContentNegotiation\ContentNegotiator;
use Aphiria\ContentNegotiation\MediaTypeFormatterMatcher;
use Aphiria\ContentNegotiation\MediaTypeFormatters\JsonMediaTypeFormatter;
use Aphiria\ContentNegotiation\MediaTypeFormatters\XmlMediaTypeFormatter;

// Register whatever media type formatters you support
$mediaTypeFormatters = [
    new JsonMediaTypeFormatter(),
    new XmlMediaTypeFormatter()
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

If you're using the <a href="https://github.com/aphiria/app/issues" target="_blank">skeleton app</a>, you don't have to worry about negotiating requests - it's done for you automatically.  If you're not using it, then let's build off of the [previous example](#basics) and negotiate a request manually.  Let's assume the raw request looked something like this:

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

Here's how we'd negotiate and deserialize the request body:

```php
use Aphiria\ContentNegotiation\NegotiatedBodyDeserializer;
use App\Users\User;

$bodyDeserializer = new NegotiatedBodyDeserializer($contentNegotiator);
// Assume the request was already instantiated
$user = $bodyDeserializer->readRequestBodyAs(User::class, $request);
echo $user->id; // 123
echo $user->email; // "foo@example.com"
```

<h2 id="negotiating-responses">Negotiating Responses</h2>

If you're using the <a href="https://github.com/aphiria/app/issues" target="_blank">skeleton app</a>, then negotiating a response is done for you automatically.  If you're not, though, you can manually negotiate a response by inspecting the `Accept`, `Accept-Charset`, and `Accept-Language` headers.  If those headers are missing, we default to using the first media type formatter that can write the response body.

Constructing a response with all the appropriate headers is a little involved when doing it manually, which is why Aphiria provides `NegotiatedResponseFactory` to handle it for you:

```php
use Aphiria\ContentNegotiation\NegotiatedResponseFactory;
use Aphiria\Net\Http\HttpStatusCode;

$responseFactory = new NegotiatedResponseFactory($contentNegotiator);
// Assume $user is a POPO User object
$response = $responseFactory->createResponse($request, HttpStatusCode::Ok, rawBody: $user);
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
use Aphiria\ContentNegotiation\ILanguageMatcher;
use Aphiria\Net\Http\Formatting\RequestParser;
use Aphiria\Net\Http\IRequest;

final class QueryStringLanguageMatcher implements ILanguageMatcher
{
    public function __construct(private RequestParser $requestParser) {}

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

Then, pass your language matcher into `ContentNegotiator`.

```php
use Aphiria\ContentNegotiation\ContentNegotiator;

$languageMatcher = new QueryStringLanguageMatcher(new RequestParser());
$contentNegotiator = new ContentNegotiator(
    // ...
    languageMatcher: $languageMatcher
);
```

<h2 id="media-type-formatters">Media Type Formatters</h2>

Media type formatters can read and write a particular data format to a stream.  You can get the media type formatter from `ContentNegotiationResult`, and use it to deserialize a request body to a particular type (`User` in this example):

```php
$mediaTypeFormatter = $result->formatter;
$mediaTypeFormatter->readFromStream($request->body, User::class);
```

Similarly, you can serialize a value and write it to the response body:

```php
$mediaTypeFormatter->writeToStream($valueToWrite, $response->body);
```

Aphiria provides the following formatters out of the box:

* `HtmlMediaTypeFormatter`
* `JsonMediaTypeFormatter`
* `PlainTextMediaTypeFormatter`
* `XmlMediaTypeFormatter`

> **Note:** `HtmlMediaTypeFormatter` and `PlainTextMediaTypeFormatter` only handle strings - they do not deal with objects or arrays.

<h3 id="customizing-deserialization">Customizing (De)Serialization</h3>

Under the hood, `JsonMediaTypeFormatter` and `XmlMediaTypeFormatter` use Symfony's <a href="https://symfony.com/doc/current/components/serializer.html" target="_blank">serialization component</a> to (de)serialize values.  Aphiria provides a <a href="https://github.com/aphiria/aphiria/blob/master/src/Framework/src/Serialization/Binders/SymfonySerializerBinder.php" target="_blank">binder</a> and some config settings in _config.php_ under `aphiria.serialization` to help you get started.  For more in-depth tutorials on how to customize Symfony's serializer, refer to <a href="https://symfony.com/doc/current/components/serializer.html" target="_blank">its documentation</a>.

<h1 id="doc-title">Testing APIs</h1>

<nav class="toc-nav">

<div class="toc-nav-contents">

<h2 id="table-of-contents">Table of Contents</h2>

<ol>
<li><a href="#introduction">Introduction</a></li>
<li><a href="#sending-requests">Sending Requests</a><ol>
<li><a href="#negotiating-content">Negotiating Content</a></li>
</ol>
</li>
<li><a href="#mocking-authentication">Mocking Authentication</a></li>
<li><a href="#response-assertions">Response Assertions</a><ol>
<li><a href="#assert-cookie-equals">assertCookieEquals</a></li>
<li><a href="#assert-has-cookie">assertHasCookie</a></li>
<li><a href="#assert-has-header">assertHasHeader</a></li>
<li><a href="#assert-header-equals">assertHeaderEquals</a></li>
<li><a href="#assert-header-matches-regex">assertHeaderMatchesRegex</a></li>
<li><a href="#assert-parsed-body-equals">assertParsedBodyEquals</a></li>
<li><a href="#assert-parsed-body-passes-callback">assertParsedBodyPassesCallback</a></li>
<li><a href="#assert-status-code-equals">assertStatusCodeEquals</a></li>
</ol>
</li>
</ol>

</div>

</nav>

<h2 id="introduction">Introduction</h2>

Integration tests are a great way to make sure your application is working end-to-end as expected.  Aphiria, with the help of <a href="https://phpunit.de/" target="_blank">PHPUnit</a>, comes with some great tools to help you send requests to your application and parse the responses.  Aphiria uses automatic [content-negotiation](content-negotiation.md) in your integration tests, which frees you to make assertions using your app's models without worrying about how to (de)serialize data.  The integration tests won't actually send a request over HTTP, either.  Instead, it creates an in-memory instance of your application, and sends the requests to that.  The nice part about not having to go over HTTP is that you don't worry about having to set up firewall rules for testing your application in staging slots.

> **Note:** All the examples here are assuming that you're using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, and extending `IntegrationTestCase`.  By default, tests will run in the `testing` environment.  If you wish to change this, you can by editing the `APP_ENV` value in _phpunit.xml.dist_.

<h2 id="sending-requests">Sending Requests</h2>

Sending a request is very simple:

```php
use App\Tests\IntegrationTestCase;
use Aphiria\Net\Http\HttpStatusCode;

final class BookQueryTest extends IntegrationTestCase
{
    public function testQueryYieldsCorrectResult(): void
    {
        $response = $this->get('/books/search?query=great%20gatsby');
        $this->assertStatusCodeEquals(HttpStatusCode::Ok, $response);
        $this->assertParsedBodyEquals(new Book('The Great Gatsby'), $response);
    }
}
```

Use the following methods to send [requests](http-requests.md) and get [responses](http-responses.md):

* `delete()`
* `get()`
* `options()`
* `patch()`
* `post()`
* `put()`

When specifying a URI, you can use either pass just the path or the fully-qualified URI.  If you're using a path, the `APP_URL` environment variable will be prepended automatically.

```php
$this->get('/books/123');

// Or

$this->get('http://localhost:8080/books/123');
```

You can pass in headers in your calls:

```php
$this->delete('/users/123', headers: ['Authorization' => 'Basic Zm9vOmJhcg==']);
```

All methods except `get()` also support passing in a body:

```php
$this->post('/users', body: new User('foo@bar.com'));
```

If you pass in an instance of `IBody`, that will be used as the request body.  Otherwise, [content negotiation](content-negotiation.md) will be applied to the value you pass in.

<h2 id="negotiating-content">Negotiating Content</h2>

You may find yourself wanting to [retrieve the response body as a typed PHP object](content-negotiation.md).  For example, let's say your API has an endpoint to create a user, and you want to make sure to delete that user by ID after your test:

```php
$response = $this->post('/users', body: new User('foo@bar.com'));
// Assume our application has a CreatedUser class with an ID property
$createdUser = $this->negotiateResponseBody(CreatedUser::class, $response);

// Perform some assertions...

$this->delete("/users/{$createdUser->id}");
```

<h2 id="mocking-authentication">Mocking Authentication</h2>

Mocking authentication calls in integration tests is easy.  Just call `actingAs()` in your test and pass in a callback for the call(s) you want to make while authenticating as the desired principal:

```php
use Aphiria\Net\Http\HttpStatusCode;
use Aphiria\Net\Http\StringBody;
use Aphiria\Security\Identity;
use Aphiria\Security\User;
use App\Tests\Integration\IntegrationTestCase;

class UserIntegrationTest extends IntegrationTestCase
{
    public function testUpdatingEmail(): void
    {
        // For the scoped call in actingAs(), we'll authenticate as the input user
        $user = new User([new Identity([])]);
        $body = new StringBody('foo@bar.com');
        $response = $this->actingAs($user, fn () => $this->put('/email', body: $body));
        $this->assertStatusCodeEquals(HttpStatusCode::Ok, $response);
    }
}
```

<h2 id="response-assertions">Response Assertions</h2>

Response assertions can be used to make sure the data sent back by your application is correct.  Let's look at some examples of assertions:

Name | Description
------ | ------
[`assertCookieEquals()`](#assert-cookie-equals) | Asserts that the response sets a cookie with a particular value
[`assertCookieIsUnset()`](#assert-has-cookie) | Asserts that the response unsets a cookie
[`assertHasCookie()`](#assert-has-cookie) | Asserts that the response sets a cookie
[`assertHasHeader()`](#assert-has-header) | Asserts that the response sets a particular header
[`assertHeaderEquals()`](#assert-header-equals) | Asserts that the response sets a header with a particular value.  Note that if you pass in a non-array value, then only the first header value is compared against.
[`assertHeaderMatchesRegex()`](#assert-header-matches-regex) | Asserts that the response sets a header whose value matches a regular expression
[`assertParsedBodyEquals()`](#assert-parsed-body-equals) | Asserts that the [parsed response body](content-negotiation.md) equals a particular value
[`assertParsedBodyPassesCallback()`](#assert-parsed-body-passes-callback) | Asserts that the [parsed response body](content-negotiation.md) passes a callback function
[`assertStatusCodeEquals()`](#assert-status-code-equals) | Asserts that the response status code equals a particular value

<h3 id="assert-cookie-equals">assertCookieEquals</h3>

```php
// Assert that the response sets a cookie named "userId" with value "123"
$this->assertCookieEquals('123', $response, 'userId');
```

<h3 id="assert-has-cookie">assertHasCookie</h3>

```php
// Assert that the response sets a cookie named "userId"
$this->assertHasCookie($response, 'userId');
```

<h3 id="assert-has-header">assertHasHeader</h3>

```php
// Assert that the response sets a header named "Authorization"
$this->assertHasHeader($response, 'Authorization');
```

<h3 id="assert-header-equals">assertHeaderEquals</h3>

```php
// Assert that the response sets a header named "Authorization" with value "Bearer abc123"
$this->assertHeaderEquals('Bearer abc123', $response, 'Authorization');
```

<h3 id="assert-header-matches-regex">assertHeaderMatchesRegex</h3>

```php
// Assert that the response sets a header named "Authorization" with a value that passes the regex
$this->assertHeaderMatchesRegex('/^Bearer [a-z0-9]+$/i', $response, 'Authorization');
```

<h3 id="assert-parsed-body-equals">assertParsedBodyEquals</h3>

```php
// Assert that the response body, after content negotiation, equals a value
$this->assertParsedBodyEquals(new User('Dave'), $response);
```

<h3 id="assert-parsed-body-passes-callback">assertParsedBodyPassesCallback</h3>

```php
// Assert that the response body, after content negotiation, passes a callback
$this->assertParsedBodyPassesCallback($response, User::class, fn ($user) => $user->name === 'Dave');
```

<h3 id="assert-status-code-equals">assertStatusCodeEquals</h3>

```php
// Assert that the response status code equals a value (can also use an int)
$this->assertStatusCodeEquals(HttpStatusCode::Ok, $response);
```

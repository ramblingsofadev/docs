<h1 id="doc-title">PSR Adapters</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Introduction](#introduction)
   1. [Why Doesn't Aphiria Adopt All The PSRs?](#why-doesnt-aphiria-adopt-all-the-psrs)
2. [PSR-7](#psr-7)
   1. [Requests](#psr-7-requests)
   1. [Responses](#psr-7-responses)
   1. [Streams](#psr-7-streams)
   1. [URIs](#psr-7-uris)
   1. [Uploaded Files](#psr-7-uploaded-files)
3. [PSR-11](#psr-11)
   1. [Creating a PSR-11 Container](#creating-a-psr-11-container)

</div>

</nav>

<h2 id="introduction">Introduction</h2>

A group called <a href="https://www.php-fig.org/" target="_blank">The PHP Framework Interop Group</a> (FIG) came together to try and solve some of the issues with reusing components between frameworks.  For example, most frameworks tend to write their own HTTP requests, responses, DI containers, etc.  Having many different implementations to solve the same thing makes it difficult to build libraries that could work across frameworks.  So, the FIG set about introducing some standards called PSRs to simplify reusing these components across frameworks.

For reasons we'll discuss in the [next section](#why-doesnt-aphiria-adopt-all-the-psrs), Aphiria does not use most PSRs natively.  However, we do offer adapters for [PSR-7](#psr-7) and [PSR-11](#psr-11) to make it possible to convert Aphiria models to PSR-compliant models and vice versa.

<h3 id="why-doesnt-aphiria-adopt-all-the-psrs">Why Doesn't Aphiria Adopt All The PSRs?</h3>

A lot of major frameworks, eg Symfony and Laravel, have also decided not to adopt some PSRs.  For example, PSR-7 has been pretty widely criticized for the following reasons:

* Requests and responses are immutable, which was seen as a poor application of immutability
  - It's all too easy to forget to get the new instance from any `with*()` methods, leading to bugs
  - Inconsistencies in some of the internals of PHP and the PSR make it impossible to achieve true immutability
* HTTP message bodies are not inherently streams - they should be readable as streams
* `ServerRequestInterface` too closely mirrors some PHP superglobals, for better or for worse
  - It relies on PHP `$_SERVER` and `$_FILES` structures, which are full of idiosyncrasies

These negatives aside, PSR-7 did do a decent job in some of its modeling of HTTP messages.  In fact, some of Aphiria's message models are somewhat similar to PSR-7, but attempt to improve their shortcomings.

PSR-11 was too limiting, eg no [targeted bindings](dependency-injection.md#targeted-bindings), and didn't provide enough value to justify supporting it.

<h2 id="psr-7">PSR-7</h2>

PSR-7 is focused on creating a standard for HTTP messages, URIs, streams, and uploaded files.  For those that wish to use third party libraries that only support PSR-7, we've created an easy way to convert to and from PSR-7 via `Psr7Factory`.

```php
use Aphiria\PsrAdapters\Psr7\Psr7Factory;
use Nyholm\Psr7\Factory\Psr17Factory;

// This third party factory can create all PSR-7 models for us
$psr17Factory = new Psr17Factory();
$psr7Factory = new Psr7Factory(
    psr7RequestFactory: $psr17Factory,
    psr7ResponseFactory: $psr17Factory,
    psr7StreamFactory: $psr17Factory,
    psr7UploadedFileFactory: $psr17Factory,
    psr7UriFactory: $psr17Factory
);
```

We'll use this factory in the examples below.

<h3 id="psr-7-requests">Requests</h3>

Create a PSR-7 request from an Aphiria request:

```php
$psr7Request = $psr7Factory->createPsr7Request($aphiriaRequest);
```

Create an Aphiria request from a PSR-7 request:

```php
$aphiriaRequest = $psr7Factory->createAphiriaRequest($psr7Request);
```

<h3 id="psr-7-responses">Responses</h3>

Create a PSR-7 response from an Aphiria response:

```php
$psr7Response = $psr7Factory->createPsr7Response($aphiriaResponse);
```

Create an Aphiria response from a PSR-7 response:

```php
$aphiriaResponse = $psr7Factory->createAphiriaResponse($psr7Response);
```

<h3 id="psr-7-streams">Streams</h3>

Create a PSR-7 stream from an Aphiria stream:

```php
$psr7Stream = $psr7Factory->createPsr7Stream($aphiriaStream);
```

Create an Aphiria stream from a PSR-7 stream:

```php
$aphiriaStream = $psr7Factory->createAphiriaStream($psr7Stream);
```

<h3 id="psr-7-uris">URIs</h3>

Create a PSR-7 URI from an Aphiria URI:

```php
$psr7Uri = $psr7Factory->createPsr7Uri($aphiriaUri);
```

Create an Aphiria URI from a PSR-7 URI:

```php
$aphiriaUri = $psr7Factory->createAphiriaUri($psr7Uri);
```

<h3 id="psr-7-uploaded-files">Uploaded Files</h3>

Create PSR-7 uploaded files from an Aphiria request:

```php
$psr7UploadedFiles = $psr7Factory->createPsr7UploadedFiles($aphiriaRequest);
```

<h2 id="psr-11">PSR-11</h2>

PSR-11's aim was to give a simple interface for [dependency injection containers](dependency-injection.md) to implement.  Although `IContainer` does not natively support PSR-11, it's trivial to create a container [that does](#creating-a-psr-11-container).

<h3 id="creating-a-psr-11-container">Creating a PSR-11 Container</h3>

```php
use Aphiria\DependencyInjection\Container;
use Aphiria\PsrAdapters\Psr11\Psr11Container;

$aphiriaContainer = new Container();
$psr11Container = new Psr11Container($aphiriaContainer);

if ($psr11Container->has(Foo::class)) {
    $foo = $psr11Container->get(Foo::class);
}
```

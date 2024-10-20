<h1 id="doc-title">Input/Output</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Streams](#streams)
   1. [Reading from a Stream](#reading-from-stream)
   2. [Writing to a Stream](#writing-to-stream)
   3. [Seeking](#seeking)
   4. [Getting the Length of a Stream](#getting-length-of-stream)
   5. [Copying to Another Stream](#copying-to-another-stream)
   6. [Closing a Stream](#closing-stream)
   7. [Multi-Streams](#multi-streams)

</div>

</nav>

<h2 id="streams">Streams</h2>

Streams allow you to read and write data in a memory-efficient way.  PHP has built-in support for streams, but the syntax is clunky, and it requires a bit of boilerplate code to work with.  Aphiria wraps PHP's streaming functionality into a simple interface: `Aphiria\IO\Streams\IStream`.  Creating a stream is easy.

```php
use Aphiria\IO\Streams\Stream;

$stream = new Stream(fopen('path/to/file', 'r+b'));
```

<h3 id="reading-from-stream">Reading from a Stream</h3>

You can read chunks of data from a stream via

```php
// Read 64 bytes from the stream
$stream->read(64);
```

You can also read to the end of a stream via

```php
$stream->readToEnd();
```

> **Note:** This will read to the end of the stream from the current cursor position.  To read the entire stream from the beginning, use `(string)$stream`.

<h3 id="writing-to-stream">Writing to a Stream</h3>

To write to a stream, call

```php
$stream->write('foo');
```

<h3 id="seeking">Seeking</h3>

To seek to a specific point in the stream, call

```php
// Seek to the 1024th byte
$stream->seek(1024);
```

To rewind to the beginning, you can call

```php
$stream->rewind();
```

<h3 id="getting-length-of-stream">Getting the Length of a Stream</h3>

To get the length of a stream, call

```php
$stream->length;
```

If it is not knowable, then `length` will return `null`.

> **Note:** If you happen to know the length of the stream ahead of time, you can pass it into the constructor, eg `new Stream(fopen('path/to/file', 'r+b'), 2056)`.  If you write anything to the stream, then the length is recalculated.

<h3 id="copying-to-another-stream">Copying to Another Stream</h3>

Sometimes, you'll need to copy one stream to another.  One example would be writing a response body's stream to the `php://output` stream.  You can do this via

```php
$destinationStream = new Stream(fopen('php://output', 'r+b'));
$sourceStream = new Stream(fopen('path/to/file', 'r+b'));
$sourceStream->copyToStream($destinationStream);
```

> **Note:** Copying to a stream does not rewind the source or destination streams.  If you want to write the entire source stream to the destination, then call `$sourceStream->rewind()` prior to `$sourceStream->copyToStream()`.

<h3 id="closing-stream">Closing a Stream</h3>

You can close a stream via

```php
$stream->close();
```

> **Note:** When PHP performs garbage collection, `close()` is automatically called by the destructor.

<h3 id="multi-streams">Multi-Streams</h3>

In some cases, such as [multi-part requests](http-requests.md#multipart-requests), you may need to append multiple streams together, yet treat them like a single stream.  This is where `MultiStream` comes in handy:

```php
use Aphiria\IO\Streams\MultiStream;
use Aphiria\IO\Streams\Stream;

$stream1 = new Stream('php://temp', 'r+b');
$stream1->write('foo');
$stream2 = new Stream('php://temp', 'r+b');
$stream2->write('bar');

$multiStream = new MultiStream();
$multiStream->addStream($stream1);
$multiStream->addStream($stream2);
echo (string)$multiStream; // "foobar"
```

<h1 id="doc-title">Input/Output</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Introduction](#introduction)
2. [File System](#file-system)
    1. [Reading a File](#reading-a-file)
    2. [Writing to a File](#writing-to-a-file)
    3. [Appending to a File](#appending-to-a-file)
    4. [Deleting a File](#deleting-a-file)
    5. [Checking if Something is a File](#checking-if-something-is-a-file)
    6. [Checking if a File is Readable](#checking-if-a-file-is-readable)
    7. [Checking if a File is Writable](#checking-if-a-file-is-writable)
    8. [Copying a File](#copying-a-file)
    9. [Moving a File](#moving-a-file)
    10. [Getting a File's Directory Name](#getting-a-files-directory-name)
    11. [Getting a File's Basename](#getting-a-files-basename)
    12. [Getting a File's Name](#getting-a-files-name)
    13. [Getting a File's Extension](#getting-a-files-extension)
    14. [Getting a File's Size](#getting-a-files-size)
    15. [Getting a File's Last Modified Time](#getting-a-files-last-modified-time)
    16. [Getting Files in a Directory](#getting-files-in-a-directory)
    17. [Getting Files with Glob](#getting-files-with-glob)
    18. [Checking if a File or Directory Exists](#checking-if-a-file-or-directory-exists)
    19. [Creating a Directory](#creating-a-directory)
    20. [Deleting a Directory](#deleting-a-directory)
    21. [Getting the List of Directories](#getting-the-list-of-directories)
    22. [Copying a Directory](#copying-a-directory)
3. [Streams](#streams)
    1. [Reading from a Stream](#reading-from-stream)
    2. [Writing to a Stream](#writing-to-stream)
    3. [Seeking](#seeking)
    4. [Getting the Length of a Stream](#getting-length-of-stream)
    5. [Copying to Another Stream](#copying-to-another-stream)
    6. [Closing a Stream](#closing-stream)
    7. [Multi-Streams](#multi-streams)

</div>

</nav>

<h2 id="introduction">Introduction</h2>

Most programs interact with a computer's file system in some way.  Aphiria comes with the `FileSystem` class to facilitate these interactions.  With it, you can easily read and write files, get attributes of files, copy files and folders, and recursively delete directories, and do other common tasks.

<h2 id="file-system">File System</h2>

For all examples below, assume `$fileSystem = new \Aphiria\IO\FileSystem();`.

<h3 id="reading-a-file">Reading a File</h3>

```php
$fileSystem->read(FILE_PATH);
```

<h3 id="writing-to-a-file">Writing to a File</h3>

```php
// The third parameter is identical to PHP's file_put_contents() flags
$fileSystem->write(FILE_PATH, 'foo', \LOCK_EX);
```

<h3 id="appending-to-a-file">Appending to a File</h3>

```php
$fileSystem->append(FILE_PATH, 'foo');
```

<h3 id="deleting-a-file">Deleting a File</h3>

```php
$fileSystem->deleteFile(FILE_PATH);
```

<h3 id="checking-if-something-is-a-file">Checking if Something is a File</h3>

```php
$fileSystem->isFile(FILE_PATH);
```

<h3 id="checking-if-a-file-is-readable">Checking if a File is Readable</h3>

```php
$fileSystem->isReadable(FILE_PATH);
```

<h3 id="checking-if-a-file-is-writable">Checking if a File is Writable</h3>

```php
$fileSystem->isWritable(FILE_PATH);
```

<h3 id="copying-a-file">Copying a File</h3>

```php
$fileSystem->copy(SOURCE_FILE, TARGET_PATH);
```

<h3 id="moving-a-file">Moving a File</h3>

```php
// This is analogous to "cutting" the file
$fileSystem->move(SOURCE_FILE, TARGET_PATH);
```

<h3 id="getting-a-files-directory-name">Getting a File's Directory Name</h3>

```php
$fileSystem->getDirectoryName(FILE_PATH);
```

<h3 id="getting-a-files-basename">Getting a File's Basename</h3>

```php
// This returns everything in the file name except for the path preceding it
$fileSystem->getBaseName(FILE_PATH);
```

<h3 id="getting-a-files-name">Getting a File's Name</h3>

```php
// This returns the file name without the extension
$fileSystem->getFileName(FILE_PATH);
```

<h3 id="getting-a-files-extension">Getting a File's Extension</h3>

```php
$fileSystem->getExtension(FILE_PATH);
```

<h3 id="getting-a-files-size">Getting a File's Size</h3>

```php
// The size of the file in bytes
$fileSystem->getFileSize(FILE_PATH);
```

<h3 id="getting-a-files-last-modified-time">Getting a File's Last Modified Time</h3>

```php
$fileSystem->getLastModified(FILE_PATH);
```

<h3 id="getting-files-in-a-directory">Getting Files in a Directory</h3>

```php
// The second parameter determines whether or not we recurse into subdirectories
// This returns the full path of all the files found
$fileSystem->getFiles(DIRECTORY_PATH, true);
```

<h3 id="getting-files-with-glob">Getting Files with Glob</h3>

```php
// See documentation for PHP's glob() function
$filesSystem->glob($pattern, $flags);
```

<h3 id="checking-if-a-file-or-directory-exists">Checking if a File or Directory Exists</h3>

```php
$fileSystem->exists(FILE_PATH);
```

<h3 id="creating-a-directory">Creating a Directory</h3>

```php
// The second parameter is the chmod permissions
// The third parameter determines whether or not we create nested subdirectories
$fileSystem->createDirectory(DIRECTORY_PATH, 0777, true);
```

<h3 id="deleting-a-directory">Deleting a Directory</h3>

```php
// The second parameter determines whether or not we keep the directory structure
$fileSystem->deleteDirectory(DIRECTORY_PATH, false);
```

<h3 id="getting-the-list-of-directories">Getting the List of Directories</h3>

```php
// The second parameter determines whether or not we recurse into the directories
// This returns the full path of all the directories found
$fileSystem->getDirectories(DIRECTORY_PATH, true);
```

<h3 id="copying-a-directory">Copying a Directory</h3>

```php
$fileSystem->copyDirectory(SOURCE_DIRECTORY, TARGET_PATH);
```

<h2 id="streams">Streams</h2>

Streams allow you to read and write data in a memory-efficient way.  They make it easy to work with large files without crippling your server.  PHP has built-in support for streams, but the syntax is clunky, and it requires a bit of boilerplate code to work with.  Aphiria wraps PHP's streaming functionality into a simple interface: `Aphiria\IO\Streams\IStream` (`Stream` comes built-in).

Here's how you can create a stream:

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
$stream->getLength();
```

If it is not knowable, then `getLength()` will return `null`.

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

In some cases, such as multi-part responses, you may need to append multiple streams together, yet treat them like a single stream.  This is where `MultiStream` comes in handy:

```php
use Aphiria\IO\Streams\MultiStream;
use Aphiria\IO\Streams\Stream;

$multiStream = new MultiStream();
$stream1 = new Stream('php://temp', 'r+');
$stream1->write('foo');
$stream2 = new Stream('php://temp', 'r+');
$stream2->write('bar');
$multiStream->addStream($stream1);
$multiStream->addStream($stream2);
echo (string)$multiStream; // "foobar"
```

<h1 id="doc-title">Reflection</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Class Finder](#class-finder)

</div>

</nav>

<h2 id="class-finder">Class Finder</h2>

PHP does not provide any native way of finding classes from files unless you've already autoloaded them into memory.  Aphiria's `FileClassFinder` can do just that, though.  You simply pass a directory or list of directories, and it will scan the directories for any classes defined within.

```php
use Aphiria\Reflection\FileClassFinder;

$classFinder = new FileClassFinder();
$classes = $classFinder->findAllClasses('PATH_TO_SCAN');
```

Or, pass in a list of directories:

```php
$classes = $classFinder->findAllClasses(['PATH1_TO_SCAN', 'PATH2_TO_SCAN']);
```

You can also recursively scan the input directories:

```php
$classes = $classFinder->findAllClasses('PATH_TO_SCAN', true);
```

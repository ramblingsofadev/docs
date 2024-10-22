<h1 id="doc-title">Reflection</h1>

<nav class="toc-nav">

<div class="toc-nav-contents">

<h2 id="table-of-contents">Table of Contents</h2>

<ol>
<li><a href="#type-finder">Type Finder</a><ol>
<li><a href="#finding-classes">Finding Classes</a></li>
<li><a href="#finding-interfaces">Finding Interfaces</a></li>
<li><a href="#finding-all-types">Finding All Types</a></li>
<li><a href="#finding-sub-types">Finding Sub-Types</a></li>
</ol>
</li>
</ol>

</div>

</nav>

<h2 id="type-finder">Type Finder</h2>

PHP does not provide any native way of finding types (eg classes and interfaces) from files unless you've already autoloaded them into memory.  Aphiria's `TypeFinder` can do just that, though.  You simply pass a directory or list of directories, and it will scan them for any types defined within.  It does this by tokenizing all PHP files and scanning for type definitions.  This is a pretty slow process, and the results should be cached whenever possible.

<h3 id="finding-classes">Finding Classes</h3>

Let's look at an example that finds all classes defined in a particular path.

```php
use Aphiria\Reflection\TypeFinder;

$typeFinder = new TypeFinder();
$classes = $typeFinder->findAllClasses('PATH_TO_SCAN');
```

Or, pass in a list of directories:

```php
$classes = $typeFinder->findAllClasses(['PATH1_TO_SCAN', 'PATH2_TO_SCAN']);
```

You can also recursively scan the input directories:

```php
$classes = $typeFinder->findAllClasses('PATH_TO_SCAN', true);
```

By default, abstract classes are not returned in the results, but you may include them:

```php
$classes = $typeFinder->findAllClasses('PATH_TO_SCAN', includeAbstractClasses: true);
```

<h3 id="finding-interfaces">Finding Interfaces</h3>

Just like you can scan for interfaces, you may also scan for interfaces.

```php
$interfaces = $typeFinder->findAllInterfaces('PATH_TO_SCAN');
```

Or recursively:

```php
$interfaces = $typeFinder->findAllInterfaces('PATH_TO_SCAN', true);
```

<h3 id="finding-all-types">Finding All Types</h3>

If you're simply trying to grab all types, regardless of whether they're a class, abstract class, or interface, there's a method for that:

```php
$types = $typeFinder->findAllTypes('PATH_TO_SCAN');
```

Or recursively:

```php
$types = $typeFinder->findAllTypes('PATH_TO_SCAN', true);
```

<h3 id="finding-sub-types">Finding Sub-Types</h3>

You can grab all sub-types of a parent type:

```php
$subTypesOfFoo = $typeFinder->findAllSubtypesOfType(Foo::class, 'PATH_TO_SCAN');
```

Or recursively:

```php
$subTypesOfFoo = $typeFinder->findAllSubtypesOfType(Foo::class, 'PATH_TO_SCAN', true);
```

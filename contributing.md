<h1 id="doc-title">Contributing</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
2. [Bugs](#bugs)
   1. [Reporting a Bug](#reporting-bug)
   2. [Fixing a Bug](#fixing-bug)
3. [Features](#features)
4. [Security Vulnerabilities](#security-vulnerabilities)
5. [Coding Style](#coding-style)
   1. [Linter](#linter)
   2. [Static Analysis](#static-analysis)
   3. [PHPDoc](#phpdoc)
6. [Naming Conventions](#naming-conventions)
   1. [Variables](#variables)
   2. [Functions/Methods](#functions-methods)
   3. [Constants](#constants)
   4. [Namespaces](#namespaces)
   5. [Classes](#classes)
   6. [Abstract Classes](#abstract-classes)
   7. [Interfaces](#interfaces)
   8. [Traits](#traits)

</div>

</nav>

<h1 id="basics">Basics</h1>

First, thank you for taking the time to contribute to Aphiria!  We use GitHub pull requests for all code contributions.  To get started on a [bug fix](#bugs) or [feature](#features), fork <a href="https://github.com/aphiria/aphiria" target="_blank">Aphiria</a>, and create a branch off of `0.x`.  Be sure to run `composer test` locally before opening the pull request to run the unit tests, [static analyzer](#static-analysis), and [linter](#linter).  Once your bug fix/feature is complete, open a pull request against `0.x`.

All pull requests **must**:

* Have 100% code coverage per PHPUnit and Xdebug 3
* Abide by our [naming conventions](#naming-conventions)
* Have no [static analysis](#static-analysis) errors
* Have no [linter](#linter) errors

<h2 id="bugs">Bugs</h2>

Before you attempt to write a bug fix, first read the documentation to see if you're perhaps using Aphiria incorrectly.  If you find a hole in our documentation, feel free to open a <a href="https://github.com/aphiria/docs" target="_blank">pull request</a> to fix it.

<h3 id="reporting-bug">Reporting a Bug</h3>

To report a bug with either the <a href="https://github.com/aphiria/aphiria/issues" target="_blank">framework</a> or <a href="https://github.com/aphiria/app/issues" target="_blank">skeleton app</a>, create a new GitHub issue with a descriptive title, steps to reproduce the bug (eg a failing PHPUnit test), and information about your environment.

<h3 id="fixing-bug">Fixing a Bug</h3>

To fix a bug, create a pull request with the fix and relevant PHPUnit tests that provide 100% code coverage.  Before opening a pull request, run `composer test` to run unit tests, the [linter](#linter), and [the static analyzer](#static-analysis).

<h2 id="features">Features</h2>

We always appreciate when you want to add a new feature to Aphiria.  For minor, backwards-compatible features, create a pull request.  Do not submit pull requests to individual libraries' repositories.  For major, possibly backwards-incompatible features, please open an issue first to discuss it prior to opening a pull request.

Aphiria strives to not create any unnecessary library dependencies.  This even includes having dependencies on other Aphiria libraries whenever possible.  If your change will introduce a new dependency to a library, create an issue and ask about it before implementing it.

<h2 id="security-vulnerabilities">Security Vulnerabilities</h2>

Aphiria takes security seriously.  If you find a security vulnerability, please email us at <a href="mailto:bugs@aphiria.com">bugs@aphiria.com</a>.

<h2 id="coding-style">Coding Style</h2>

Aphiria follows <a href="http://www.php-fig.org/psr/psr-1/" title="PSR-1 spec" target="_blank">PSR-1</a>, <a href="http://www.php-fig.org/psr/psr-2/" title="PSR-2 spec" target="_blank">PSR-2</a>, and  <a href="http://www.php-fig.org/psr/psr-12/" title="PSR-12 spec" target="_blank">PSR-12</a> coding standards and uses <a href="http://www.php-fig.org/psr/psr-4/" title="PSR-4 spec" target="_blank">PSR-4</a> autoloading.  All PHP files should specify `declare(strict_types=1);`.  Additionally, unless a class is specifically meant to be extended, declare them as `final` to encourage composition over inheritance.

<h3 id="linter">Linter</h3>

All code is run through <a href="https://github.com/FriendsOfPHP/PHP-CS-Fixer" target="_blank">PHP-CS-Fixer</a>, a powerful linter.  Pull requests that do not pass the linter will automatically be prevented from being merged.  You can run the linter locally via `composer phpcs-test` to check for errors, and `composer phpcs-fix` to fix any errors.

<h3 id="static-analysis">Static Analysis</h3>

Aphiria uses the terrific static analysis tool <a href="https://psalm.dev" target="_blank">Psalm</a>.  It can detect things like unused code, inefficient code, and incorrect types.  We use the highest level of error reporting.  You can run Psalm locally via `composer psalm`.  Occasionally, Psalm might highlight false positives, which can be suppressed with `/** @psalm-suppress {issue handler name} {brief description of why you're suppressing it} */` immediately before the problematic line.  You can also suppress false positives in _psalm.xml.dist_ to suppress them at the directory- and file-levels.  Be sure to include an XML comment (eg `<!-- {brief description of why you're suppressing it} -->`) if you're suppressing errors in _psalm.xml.dist_.  Use error suppression sparingly - try to fix any legitimate issues that Psalm finds.

<h3 id="phpdoc">PHPDoc</h3>

Use PHPDoc to document **all** class properties, methods, and functions.  Constructors only need to document the parameters.  Method/function PHPDoc must include one blank line between the description and the following tag.  Here's an example:

```php
final class Book
{
    /**
     * @param string $title The title of the book
     */
    public function __construct(private string $title)
    {
        $this->setTitle($title);
    }
    
    /**
     * Gets the title of the book
     *
     * @return string The title of the book
     */
    public function getTitle(): string
    {
        return $this->title;
    }
    
    /**
     * Sets the title of the book
     *
     * @param string $title The title of the book
     * @return self For object chaining
     */
    public function setTitle(string $title): self
    {
        $this->title = $title;
        
        return $this;
    }
}
```

<h2 id="naming-conventions">Naming Conventions</h2>

Inspired by <a href="http://www.amazon.com/Code-Complete-Practical-Handbook-Construction/dp/0735619670" target="_blank">Code Complete</a>, Aphiria uses a straightforward approach to naming things.

<h3 id="variables">Variables</h3>

All variable names:

* Must be lower camel case, eg `$emailAddress`
* Must not use Hungarian Notation, eg `$arrUsers`

<h3 id="functions-methods">Functions/Methods</h3>

All function/method names:

* Must be succinct
  * Your method name should describe exactly what it does, nothing more, and nothing less
  * If you are having trouble naming a method, that's probably a sign it is doing too much and should be refactored
* Must be lower camel case, eg `compileList()`
  * Acronyms in function/method names &le; 2 characters long, capitalize each character, eg `startIO()`
  * "Id" is an abbreviation (not an acronym) for "Identifier", so it should be capitalized `Id`
* Must answer a question if returning a boolean variable, eg `hasAccess()` or `userIsValid()`
  * Always think about how your function/method will be read aloud in an `if` statement.  `if (userIsValid())` reads better than `if (isUserValid())`.
* Must use `getXXX()` and `setXXX()` for functions/methods that get and set properties, respectively
  * Don't name a method that returns a username `username()`.  Name it `getUsername()` so that its purpose is unambiguous.

<h3 id="constants">Constants</h3>

All class constants' names:

* Must be upper snake case, eg `TYPE_SUBSCRIBER`

<h3 id="namespaces">Namespaces</h3>

All namespaces:

* Must be Pascal case, eg `Aphiria\FooBar`
  * For namespace acronyms &le; 2 characters long, capitalize each character, eg `IO`

<h3 id="classes">Classes</h3>

All class names:

* Must be succinct
  * Your class name should describe exactly what it does, nothing more, and nothing less
  * If you are having trouble naming a class, that's probably a sign that it is doing too much and should be refactored
* Must be Pascal case, eg `ListCompiler`
  * For class name acronyms &le; 2 characters long, capitalize each character, eg `IO`
  * Class filenames should simply be the class name with *.php* appended, eg *ListCompiler.php*
    
Whenever possible, <a href="https://wiki.php.net/rfc/constructor_promotion" target="_blank">constructor property promotion</a> should be used for properties that have no custom logic in the constructor.
  
Class properties should appear before any methods.  The following is the preferred ordering of class properties and methods:

<h4 id="properties">Properties</h4>

1. Constants
2. Public static properties
3. Public properties
4. Protected static properties
5. Protected properties
6. Private static properties
7. Private properties

<h4 id="methods">Methods</h4>

1. Magic methods
2. Public static methods
3. Public abstract methods
4. Public methods
5. Protected static methods
6. Protected abstract methods
7. Protected methods
8. Private static methods
9. Private methods

> **Note:** Methods of the same visibility should be ordered alphabetically.

<h3 id="abstract-classes">Abstract Classes</h3>

All abstract class names:

* Must be Pascal case, eg `ConnectionPool`
* Must not use `Abstract`, `Base`, or any other word in the name that implies it is an abstract class
  
<h3 id="interfaces">Interfaces</h3>

All interface names:

* Must be preceded by an `I`, eg `IUser`

<h3 id="traits">Traits</h3>

All trait names:

* Must be Pascal case, eg `ListValidator`
* Must be not use `T`, `Trait`, or any other word in the name that implies it is a trait

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
   2. [Properties](#properties)
   3. [Functions/Methods](#functions-methods)
   4. [Constants](#constants)
   5. [Namespaces](#namespaces)
   6. [Classes](#classes)
   7. [Abstract Classes](#abstract-classes)
   8. [Interfaces](#interfaces)
   9. [Traits](#traits)
   10. [Enums](#enums)
7. [Financial Support](#financial-support)

</div>

</nav>

<h1 id="basics">Basics</h1>

First, thank you for taking the time to contribute to Aphiria!  We use GitHub pull requests for all code contributions.  To get started on a [bug fix](#bugs) or [feature](#features), fork <a href="https://github.com/aphiria/aphiria" target="_blank">Aphiria</a>, and create a branch off of `1.x`.  Be sure to run `composer test` locally before opening the pull request to run the unit tests, [static analyzer](#static-analysis), and [linter](#linter).  Once your bug fix/feature is complete, open a pull request against `1.x`.

All pull requests **must**:

* Have 100% code coverage per <a href="https://phpunit.de/" target="_blank">PHPUnit</a> and <a href="https://xdebug.org/" target="_blank">Xdebug 3</a>
* Abide by our [naming conventions](#naming-conventions)
* Have no [static analysis](#static-analysis) errors
* Have no [linter](#linter) errors

<h2 id="bugs">Bugs</h2>

Before you attempt to write a bug fix, first read the documentation to see if you're perhaps using Aphiria incorrectly.  If you find a hole in our documentation, feel free to open a <a href="https://github.com/aphiria/docs" target="_blank">pull request</a> to fix it.

<h3 id="reporting-bug">Reporting a Bug</h3>

To report a bug with either the <a href="https://github.com/aphiria/aphiria/issues" target="_blank">framework</a> or <a href="https://github.com/aphiria/app/issues" target="_blank">skeleton app</a>, create a new GitHub issue with a descriptive title, steps to reproduce the bug (eg a failing PHPUnit test), and information about your environment.  If you are just looking for general help, use <a href="https://github.com/aphiria/aphiria/discussions" target="_blank">GitHub Discussions</a>.

<h3 id="fixing-bug">Fixing a Bug</h3>

To fix a bug, create a pull request with the fix and relevant PHPUnit tests that provide 100% code coverage.  Before opening a pull request, run `composer test` to run unit tests, the [linter](#linter), and [the static analyzer](#static-analysis).

<h2 id="features">Features</h2>

We always appreciate when you want to add a new feature to Aphiria.  For minor, backwards-compatible features, create a pull request.  Do not submit pull requests to individual libraries' repositories.  For major, possibly backwards-incompatible features, please open an issue first to discuss it prior to opening a pull request.

Aphiria strives to not create any unnecessary library dependencies.  This even includes having dependencies on other Aphiria libraries whenever possible.  If your change will introduce a new dependency to a library, create an issue and ask about it before implementing it.

<h2 id="security-vulnerabilities">Security Vulnerabilities</h2>

Aphiria takes security seriously.  If you find a security vulnerability, please email us at <a href="mailto:bugs@aphiria.com">bugs@aphiria.com</a>.

<h2 id="coding-style">Coding Style</h2>

Aphiria follows <a href="http://www.php-fig.org/psr/psr-12/" title="PSR-12 spec" target="_blank">PSR-12</a> coding standards and uses <a href="http://www.php-fig.org/psr/psr-4/" title="PSR-4 spec" target="_blank">PSR-4</a> autoloading.  All PHP files should specify `declare(strict_types=1);`.  Additionally, unless a class is specifically meant to be extended, declare them as `final` to encourage composition over inheritance.

<h3 id="linter">Linter</h3>

All code is run through <a href="https://github.com/FriendsOfPHP/PHP-CS-Fixer" target="_blank">PHP-CS-Fixer</a>, a powerful linter.  Pull requests that do not pass the linter will automatically be prevented from being merged.  You can run the linter locally via `composer phpcs-test` to check for errors, and `composer phpcs-fix` to fix any errors.

<h3 id="static-analysis">Static Analysis</h3>

Aphiria uses the terrific static analysis tool <a href="https://psalm.dev" target="_blank">Psalm</a>.  It can detect things like unused code, inefficient code, and incorrect types.  We use the highest level of error reporting.  You can run Psalm locally via `composer psalm`.

Occasionally, Psalm might highlight false positives, which can be suppressed with:

```php
/** @psalm-suppress {issue handler name} {brief description of why you're suppressing it} */
// Problematic code here...
``` 

You can also suppress false positives in _psalm.xml.dist_ at the directory- and file-levels.  Be sure to include an XML comment explaining why the errors should be suppressed:

```xml
<issueHandlers>
    <MixedAssignment>
        <errorLevel type="suppress">
            <!-- We don't care about mixed assignments in tests -->
            <directory name="src/**/tests" />
        </errorLevel>
    </MixedAssignment>
</issueHandlers>
```

Use error suppression sparingly - try to fix any legitimate issues that Psalm finds.

<h3 id="phpdoc">PHPDoc</h3>

Use PHPDoc to document **all** class properties, methods, and functions.  Constructors only need to document the parameters.  Method/function PHPDoc must include one blank line between the description and the following tag.  Here's an example:

```php
final class User
{
    /**
     * @param string $firstName The user's first name
     * @param string $lastName The user's last name
     */
    public function __construct(
        public readonly string $firstName,
        public readonly string $lastName
    ) {
    }
    
    /**
     * Gets the user's full name
     *
     * @return string The user's full name
     */
    public function getFullName(): string
    {
        return "{$this->firstName} {$this->lastName}";
    }
}
```

<h2 id="naming-conventions">Naming Conventions</h2>

Inspired by <a href="http://www.amazon.com/Code-Complete-Practical-Handbook-Construction/dp/0735619670" target="_blank">Code Complete</a>, Aphiria uses a straightforward approach to naming things.

<h3 id="variables">Variables</h3>

All variable names:

* Must be lower camel case, eg `$emailAddress`
* Must not use Hungarian Notation, eg `$arrUsers`

<h3 id="properties">Properties</h3>

We should favor using class properties over `getXxx()` and `setXxx()` whenever accessing and setting the value is straight forward.  If a class property should only be read, it should be declared as `readonly` rather than using a `getXxx()` method.  For example, here is what **not** to do:

```php
class Book
{
    /** @var string The book title */
    private string $title;
    
    /**
     * Gets the book title
     * 
     * @return string The book title
     */
    public function getTitle(): string
    {
        return $this->title;
    }
}
```

Instead, declare the title to be `readonly`:

```php
class Book
{
    /**
     * @param string $title The book title
     */
    public function __construct(public readonly string $title)
    {
    }
}
```

The exception to this rule is when an interface needs to expose accessors to a property, but only because PHP interfaces do not support properties.

We should also favor marking properties as `readonly`, even when `private`, if their values should not be set/changed outside of the construtor.

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

<h3 id="enums">Enums</h3>

All enums:

* Must use singular names, eg `StatusCode` instead of `StatusCodes`
* Must be Pascal case, eg `StatusCode::NotFound` instead of `StatusCode::NOT_FOUND`

<h2 id="financial-support">Financial Support</h2>

Aphiria is primarily written by David Young in his spare time.  It is the labor of over a thousand hours of meticulously crafting its syntax, designing its architecture, and writing its code.  While Aphiria is and always will be free and open source, <a href="https://github.com/sponsors/davidbyoung" target="_blank">GitHub sponsorship</a> is always welcome.  While you're at it, consider sponsoring some others whose tools you might already be using:

* <a href="https://github.com/sponsors/sebastianbergmann" target="_blank">Sebastian Bergmann (PHPUnit)</a>
* <a href="https://github.com/sponsors/derickr" target="_blank">Derick Rethans (Xdebug)</a>
* <a href="https://github.com/sponsors/Seldaek" target="_blank">Jordi Boggiano (Composer, Monolog, and Symfony)</a>
* <a href="https://github.com/sponsors/fabpot" target="_blank">Fabien Potencier (Symfony)</a>
* <a href="https://github.com/sponsors/taylorotwell" target="_blank">Taylor Otwell (Laravel)</a>
* <a href="https://github.com/sponsors/keradus" target="_blank">Dariusz Rumiński (PHP-CS-Fixer and PHP Coveralls)</a>

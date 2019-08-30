# Contributing

## Table of Contents
1. [Bugs](#bugs)
   1. [Reporting a Bug](#reporting-bug)
   2. [Fixing a Bug](#fixing-bug)
2. [Features](#features)
3. [Security Vulnerabilities](#security-vulnerabilities)
4. [Coding Style](#coding-style)
   1. [PHPDoc](#phpdoc)
5. [Naming Conventions](#naming-conventions)
   1. [Variables](#variables)
   2. [Functions/Methods](#functions-methods)
   3. [Constants](#constants)
   4. [Namespaces](#namespaces)
   5. [Classes](#classes)
   6. [Abstract Classes](#abstract-classes)
   7. [Interfaces](#interfaces)
   8. [Traits](#traits)

<h2 id="bugs">Bugs</h2>

Before you attempt to write a bug fix, first read the documentation to see if you're perhaps using Aphiria incorrectly.

<h3 id="reporting-bug">Reporting a Bug</h3>

To report a bug, <a href="https://github.com/aphiria" target="_blank">create a new issue</a> for the appropriate library with a descriptive title, steps to reproduce the bug (eg a failing PHPUnit test), and information about your environment.

<h3 id="fixing-bug">Fixing a Bug</h3>

To fix a bug, create a pull request on the latest stable branch (`1.0`) of the <a href="https://github.com/aphiria" title="Aphiria repositories" target="_blank">library's repository</a> with the fix and relevant PHPUnit tests.

<h2 id="features">Features</h2>

We always appreciate when you want to add a new feature to Aphiria.  For minor, backwards-compatible features, create a pull request on the latest stable branch (`1.0`) of the <a href="https://github.com/aphiria" title="Aphiria repositories" target="_blank">library's repository</a>.  Do not submit pull requests to individual libraries' repositories.  For major, possibly backwards-incompatible features, create a pull request on the `develop` branch.  All new features should come with PHPUnit tests proving their functionality.  Pull requests should never be sent to the `master` branch.

Aphiria strives to not create any unnecessary library dependencies.  This even includes having dependencies on other Aphiria libraries.  If your change will introduce a new dependency to a library, create an issue and ask about it before implementing it.

<h2 id="security-vulnerabilities">Security Vulnerabilities</h2>

Aphiria takes security seriously.  If you find a security vulnerability, please email us at <a href="mailto:admin@aphiria.com">bugs@aphiria.com</a>.

<h2 id="coding-style">Coding Style</h2>

Aphiria follows <a href="http://www.php-fig.org/psr/psr-1/" title="PSR-1 spec" target="_blank">PSR-1</a> and <a href="http://www.php-fig.org/psr/psr-2/" title="PSR-2 spec" target="_blank">PSR-2</a> coding standards and uses <a href="http://www.php-fig.org/psr/psr-4/" title="PSR-4 spec" target="_blank">PSR-4</a> autoloading.  It uses PHP CS Fixer to enforce code style (available by running `composer run-script lint-fix` from the terminal).

All PHP files should specify `declare(strict_types=1);`.  Additionally, unless a class is specifically meant to be extended, declare them as `final` to encourage composition instead of inheritance.

<h3 id="phpdoc">PHPDoc</h3>

Use PHPDoc to document **all** class properties, methods, and functions.  Constructors only need to document the parameters.  Method/function PHPDoc must include one blank line between the description and the following tag.  Here's an example:

```php
final class Book
{
    /** @var string The title of the book */
    private string $title;
    
    /**
     * @param string $title The title of the book
     */
    public function __construct(string $title)
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
     * @return $this For object chaining
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
* Must NOT use Hungarian Notation

<h3 id="functions-methods">Functions/Methods</h3>

All function/method names:

* Must be succinct
  * Your method name should describe exactly what it does, nothing more, and nothing less
  * If you are having trouble naming a method, that's probably a sign that it is doing too much and should be refactored
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
* Must NOT use `Abstract`, `Base`, or any other word in the name that implies it is an abstract class
  
<h3 id="interfaces">Interfaces</h3>

All interface names:

* Must be preceded by an `I`, eg `IUser`

<h3 id="traits">Traits</h3>

All trait names:

* Must be Pascal case, eg `ListValidator`
* Must be NOT use `T`, `Trait`, or any other word in the name that implies it is a trait

<h1 id="doc-title">Framework Comparison</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Preface](#preface)
2. [General](#general)
   1. [Aphiria](#aphiria-general)
   2. [Symfony](#symfony-general)
   3. [Laravel](#laravel-general)
2. [Routing](#routing)
   1. [Aphiria](#aphiria-routing)
   2. [Symfony](#symfony-routing)
   3. [Laravel](#laravel-routing)
3. [Controllers](#controllers)
   1. [Aphiria](#aphiria-controllers)
   2. [Symfony](#symfony-controllers)
   3. [Laravel](#laravel-controllers)
4. [HTTP Library](#http-library)
   1. [Aphiria](#aphiria-http-library)
   2. [Symfony](#symfony-http-library)
   3. [Laravel](#laravel-http-library)
5. [Dependency Injection Container](#di-container)
   1. [Aphiria](#aphiria-di-container)
   2. [Symfony](#symfony-di-container)
   3. [Laravel](#laravel-di-container)
6. [Console](#console)
   1. [Aphiria](#aphiria-console)
   2. [Symfony](#symfony-console)
   3. [Laravel](#laravel-console)

</div>

</nav>

<h2 id="preface">Preface</h2>

Before we get into comparing Aphiria against other frameworks, remember that each framework is just a tool.  Some tools are better suited to some problems than others.  This sort of comparison is inherently subjective, although we will do our best to keep things objective.  If you feel we've misconstrued or missed anything, please feel free to submit an issue or pull request to the documentation to improve it.

<h2 id="general">General</h2>

Before we get into library comparisons, let's compare the frameworks at a high level.

<h3 id="aphiria-general">Aphiria</h3>

<h4 id="aphiria-general-pros">Pros</h4>

* Expressive syntax for building REST APIs
* Unopinionated, ie it doesn't prescribe _how_ to do something - it places that completely in the developers' hands (akin to an Android phone)
* Favors code-based configuration over magic string-based configuration
* Can easily convert to-and-from PSRs
* Thorough, searchable documentation

<h4 id="aphiria-general-cons">Cons</h4>

* Little community support
* Does not natively use some PSRs
* No built-in support for background processing of data

<h3 id="symfony-general">Symfony</h3>

<h4 id="symfony-general-pros">Pros</h4>

* An established history and huge community support
* Lots of functionality baked into the framework, eg eventing
* Some of its libraries are the back-bone of other very popular libraries and frameworks

<h4 id="symfony-general-cons">Cons</h4>

* Configuration is usually string-based
* Perceived by some to be difficult to get up and running with
* The documentation is difficult to navigate through

<h3 id="laravel-general">Laravel</h3>

<h4 id="laravel-general-pros">Pros</h4>

* Lots of community support and tutorials
* Arguably one of the easiest frameworks to get up and running with
* Lots of functionality baked into the framework
* Thorough, searchable documentation

<h4 id="laravel-general-cons">Cons</h4>

* Pretty opinionated, eg it works best if used as designed (akin to an iPhone)
* Some complain that there is too much "magic", eg unintuitive logic that powers some of its core libraries

<h2 id="routing">Routing</h2>

Routing is how you map a URI to an action, frequently a controller method.

<h3 id="aphiria-routing">Aphiria</h3>

<h4 id="aphiria-routing-pros">Pros</h4>

* Fluent syntax makes defining routes very simple
* Supports defining routes via attributes
* Supports custom constraints, which make it possible to do things like versioned endpoints
* Trie-based solution makes it one of the fastest routing libraries out there
* Not tied to any HTTP or middleware library
* Supports host and path matching

<h4 id="aphiria-routing-cons">Cons</h4>

* Does not support closure actions

<h3 id="symfony-routing">Symfony</h3>

<h4 id="symfony-routing-pros">Pros</h4>

* Supports defining routes via attributes
* Currently the fastest PHP routing library out there (barely edges out Aphiria)
* Supports host and path matching
* Supports URL generation

<h4 id="symfony-routing-cons">Cons</h4>

* Config-based route definitions are not easy to write without IDE plugins
* Middleware support is tightly coupled to Symfony HTTP and middleware libraries
* Does not support closure actions

<h3 id="laravel-routing">Laravel</h3>

<h4 id="laravel-routing-pros">Pros</h4>

* Intuitive syntax
* Supports URL generation
* Supports closure actions, although not when using route caching

<h4 id="laravel-routing-cons">Cons</h4>

* No support for custom constraints
* Route middleware are tightly coupled to Laravel's HTTP and middleware libraries

<h2 id="controllers">Controllers</h2>

Controllers are the actions that are executed when a user hits a URI.  They typically encapsulate all HTTP application logic, and transform it to and from the domain logic.

<h3 id="aphiria-controllers">Aphiria</h3>

<h4 id="aphiria-controllers-pros">Pros</h4>

* Automatic content negotiation for request and response bodies
* Automatic validation of request bodies
* Automatic deserialization of route and query string parameters
* Expressive syntax that makes your controllers more intuitive to read and write
* Convenience methods for constructing various responses

<h4 id="aphiria-controllers-cons">Cons</h4>

* Works best with REST API requests/responses, not frontend endpoints

<h3 id="symfony-controllers">Symfony</h3>

<h4 id="symfony-controllers-pros">Pros</h4>

* Provides many baked-in convenience methods for constructing responses
* Automatic deserialization of route and query string parameters
* Works well for REST API and frontend endpoints

<h4 id="symfony-controllers-cons">Cons</h4>

* No automatic content negotiation

<h3 id="laravel-controllers">Laravel</h3>

<h4 id="laravel-controllers-pros">Pros</h4>

* Has many functions that make it easy to construct responses
* Has built-in support for automatically creating controller methods for various HTTP methods
* Works well for REST API and frontend endpoints

<h4 id="laravel-controllers-cons">Cons</h4>

* Only supports negotiating Eloquent models automatically, and does not support all standard means of content negotiation

<h2 id="http-library">HTTP Library</h2>

PHP's abstractions around HTTP requests and responses are pretty bare bones.  Most frameworks find it necessary to build out classes and helper to better construct and read from requests and responses.

<h3 id="aphiria-http-library">Aphiria</h3>

<h4 id="aphiria-http-library-pros">Pros</h4>

* Clean abstractions of HTTP messages
* Helper classes to simplify common tasks with messages, eg setting cookies, determining if a request is JSON, constructing multi-part requests, etc
* Contains support for content negotiation and media type formatters
* Can convert to-and-from PSR-7

<h4 id="aphiria-http-library-cons">Cons</h4>

* Does not use PSR-7 natively
* Does not contain an HTTP client

<h3 id="symfony-http-library">Symfony</h3>

<h4 id="symfony-http-library-pros">Pros</h4>

* Contains an HTTP client to send and receive messages
* Can convert to-and-from PSR-7

<h4 id="symfony-http-library-cons">Cons</h4>

* Requires additional libraries for handling things like reading and writing cookies easily
* Does not use PSR-7 natively

<h3 id="laravel-http-library">Laravel</h3>

<h4 id="laravel-http-library-pros">Pros</h4>

* Lots of convenience methods for common tasks
* Can convert to-and-from PSR-7 because it extends Symfony's HTTP library

<h4 id="laravel-http-library-cons">Cons</h4>

* Not necessarily a con, but trades off proper abstractions for convenience
* Does not use PSR-7 natively

<h2 id="di-container">Dependency Injection Container</h2>

A dependency injection (DI) container lets a developer tell the application "When you need dependency IFoo, use this instance of IFoo".  On top of that, many DI containers support auto-wiring, which is the process of reflecting a class constructor and resolving all the parameters recursively so that the container can automatically instantiate the class.

<h3 id="aphiria-di-container">Aphiria</h3>

<h4 id="aphiria-di-container-pros">Pros</h4>

* Support for auto-wiring
* Supports binders for registering dependencies for modules
* Supports automatic lazy execution of binders so that binders that aren't used are not executed
* Straightforward methods to bind and resolve dependencies from the DI Container
* Supports targeted bindings
* Can convert to PSR-11

<h4 id="aphiria-di-container-cons">Cons</h4>

* Does not use PSR-11 natively

<h3 id="symfony-di-container">Symfony</h3>

<h4 id="symfony-di-container-pros">Pros</h4>

* Support for auto-wiring
* One of the fastest DI container libraries

<h4 id="symfony-di-container-cons">Cons</h4>

* Although it supports code-based binding of services, it mostly relies on string-based configuration
* Must manually mark services as lazy

<h3 id="laravel-di-container">Laravel</h3>

<h4 id="laravel-di-container-pros">Pros</h4>

* Support for auto-wiring
* Supports service providers for registering dependencies for modules
* Supports targeted (contextual) bindings
* Supports PSR-11 natively

<h4 id="laravel-di-container-cons">Cons</h4>

* Must manually mark service providers as deferred

<h2 id="console">Console</h2>

A console library permits a user to enter a command via a console application and map that command to an action in the code base.

<h3 id="aphiria-console">Aphiria</h3>

<h4 id="aphiria-console-pros">Pros</h4>

* Trivial to get up and running on its own
* Provides support for output formatting, eg padding/table/progress bar formatting
* Support for password masking
* Support for attribute-based commands

<h4 id="aphiria-console-cons">Cons</h4>

* Little community support

<h3 id="symfony-console">Symfony</h3>

<h4 id="symfony-console-pros">Pros</h4>

* Easy to get up and running on its own
* Provides support for output formatting, eg padding/table/progress bar formatting
* Supports password masking
* Powers other popular console libraries, even Laravel's

<h4 id="symfony-console-cons">Cons</h4>

* No built-in support for command attributes

<h3 id="laravel-console">Laravel</h3>

<h4 id="laravel-console-pros">Pros</h4>

* Integration for background running of console commands
* Auto-completion for command input

<h4 id="laravel-console-cons">Cons</h4>

* No built-in support for command attributes

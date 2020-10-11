<h1 id="doc-title">Validation</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Introduction](#introduction)
2. [Validating Data](#validating-data)
   1. [Validating Objects](#validating-objects)
   2. [Validating Properties](#validating-properties)
   3. [Validating Methods](#validating-methods)
   4. [Validating Values](#validating-values)
3. [Constraints](#constraints)
   1. [Built-In Constraints](#built-in-constraints)
   2. [Custom Constraints](#custom-constraints)
4. [Error Messages](#error-messages)
   1. [Error Message Templates](#error-message-templates)
   2. [Built-In Error Message Interpolators](#built-in-error-message-interpolators)
5. [Validation Attributes](#validation-attributes)
   1. [Built-In Attributes](#built-in-attributes)
   2. [Using Attributes](#using-attributes)
6. [Validating Request Bodies](#validating-request-bodies)

</div>

</nav>

<h2 id="introduction">Introduction</h2>

Validating your data, especially input, is critical for ensuring that your application runs smoothly.  Let's take a look at how you can do this with your POPOs in Aphiria.  Assume we have the following model in your application:

```php
final class User
{
    public function __construct(
        public int $id, 
        public string $email, 
        public string $name
    ) {}
}
```

Let's set up some constraints.

```php
use Aphiria\Validation\Builders\ObjectConstraintsRegistryBuilder;
use Aphiria\Validation\Constraints\{EmailConstraint, RequiredConstraint};
use Aphiria\Validation\Validator;
use App\Users\User;

// Set up our validator
$constraintsBuilder = new ObjectConstraintsRegistryBuilder();
$constraintsBuilder->class(User::class)
    ->hasPropertyConstraints('email', new EmailConstraint())
    ->hasPropertyConstraints('name', new RequiredConstraint());
$validator = new Validator($constraintsBuilder->build());

// Let's validate
$user = new User(123, 'dave@example.com', 'Dave');
$validator->validateObject($user);
```

If the object was not valid, a `ValidationException` will be thrown.  That's it - validation, made simple.

<h2 id="validating-data">Validating Data</h2>

Several types of data can be validated:

* [Objects](#validating-objects)
* [Properties](#validating-properties)
* [Methods](#validating-methods)
* [Raw values](#validating-values)

<h3 id="validating-objects">Validating Objects</h3>

To validate an object, simply map the properties and methods in that object to constraints.  Aphiria will then recursively validate the object and any properties/methods that contain objects.  Use `ObjectConstraintsRegistryBuilder` to help set up the constraints on your objects' properties/methods [like in the above example](#introduction).  To validate an object, we have two options:

```php
$blogPost = new BlogPost('How to Reticulate Splines');

// Will throw a ValidationException if $blogPost is invalid
$valdiator->validateObject($blogPost);

// Or

$violations = [];

// Will return true if $blogPost is valid, otherwise false
if ($validator->tryValidateObject($blogPost, $violations)) {
    // ...
}
```

<h3 id="validating-properties">Validating Properties</h3>

You can validate an individual property from an object:

```php
$blogPost = new BlogPost('How to Reticulate Splines');

// Will throw a ValidationException if $blogPost->title is invalid
$valdiator->validateProperty($blogPost, 'title');

// Or

$violations = [];

// Will return true if $blogPost->title is valid, otherwise false
if ($validator->tryValidateProperty($blogPost, 'title', $violations)) {
    // ...
}
```

In the case that the property holds an object value, it will be recursively validated, too.

<h3 id="validating-methods">Validating Methods</h3>

You can validate an individual method very similarly to how you validate properties:

```php
$blogPost = new BlogPost('How to Reticulate Splines');

// Will throw a ValidationException if $blogPost->getTitleSlug() is invalid
$valdiator->validateMethod($blogPost, 'getTitleSlug');

// Or

$violations = [];

// Will return true if $blogPost->getTitleSlug() is valid, otherwise false
if ($validator->tryValidateMethod($blogPost, 'getTitleSlug', $violations)) {
    // ...
}
```

In the case that the method holds an object value, it will also be recursively validated.

<h3 id="validating-values">Validating Values</h3>

If you want to validate an individual value, you can:

```php
// Will throw a ValidationException if $email is invalid
$valdiator->validateValue($email, [new EmailConstraint()]);

// Or

$violations = [];

// Will return true if $email is valid, otherwise false
if ($validator->tryValidateValue($email, [new EmailConstraint()], $violations)) {
    // ...
}
```

<h2 id="constraints">Constraints</h2>

A constraint is something that a value must pass to be considered valid.  For example, if you want a value to only contain alphabet characters, you can enforce the `AlphaConstraint` on it.  All constraints must implement `IConstraint`.

<h3 id="built-in-constraints">Built-In Constraints</h3>

Aphiria comes with some useful constraints built-in:

Name | Description
------ | ------
`AlphaConstraint` | The value must only contain alphabet characters
`AlphanumericConstraint` | The value must only contain alphanumeric characters
`BetweenConstraint` | The value must fall in between two values (takes in whether or not the min and max are inclusive)
`CallbackConstraint` | The value must satisfy a callback that returns a boolean
`DateConstraint` | The value must match a date-time format
`EachConstraint` | The value must satisfy a list of constraints (takes in a list of `IConstraint`)
`EqualsConstraint` | The value must equal a value
`InConstraint` | The value must be in a list of acceptable values
`IntegerConstraint` | The value must be an integer
`IPAddressConstraint` | The value must be an IP address
`MaxConstraint` | The value cannot exceed a max value (takes in whether or not the max is inclusive)
`MinConstraint` | The value cannot go below a min value (takes in whether or not the min is inclusive)
`NotInConstraint` | The value must not be in a list of values
`NumericConstraint` | The value must be numeric
`RegexConstraint` | The value must satisfy a regular expression
`RequiredConstraint` | The value must not be null

<h3 id="custom-constraints">Custom Constraints</h3>

Creating a custom constraint is simple - just implement `IConstraint`.

```php
use Aphiria\Validation\Constraints\IConstraint;

final class MaxLengthConstraint implements IConstraint
{
    public function __construct(private int $maxLength) {}

    public function getErrorMessageId(): string
    {
        return 'Length cannot exceed {maxLength}';
    }

    public function getErrorMessagePlaceholders($value): array
    {
        return ['maxLength' => $this->maxLength];
    }

    public function passes($value): bool
    {
        if (!is_string($value)) {
            throw new \InvalidArgumentException('Value must be string');
        }

        return \mb_strlen($value) <= $this->maxLength;
    }
}
```

You can now use this constraint just like any other built-in constraint:

```php
$constraintsBuilder->class(BlogPost::class)
    ->hasPropertyConstraints('title', new MaxLengthConstraint(32));
```

<h2 id="error-messages">Error Messages</h2>

Error messages provide human-readable explanations of what failed during validation.  `IConstraints` contain error message IDs and placeholders, which can give more specifics on why a constraint failed.  For example, `MaxConstraint` has a default error message ID of `Field must be less than {max}`, and it provides a `max` error message placeholder so that you can display the actual max in the error message.

Depending on how you're validating a value, there are different ways of grabbing the constraint violations.  If you're using `IValidator::validate*()` methods, you can grab the violations from the `ValidationException`:

```php
use Aphiria\Validation\ErrorMessages\StringReplaceErrorMessageInterpolator;
use Aphiria\Validation\{ValidationException, Validator};

// Assume we already have our object constraints configured
$errorMessageInterpolator = new StringReplaceErrorMessageInterpolator();
$validator = new Validator($objectConstraints, $errorMessageInterpolator);

try {
    $validator->validateObject($blogPost);
} catch (ValidationException $ex) {
    $errors = [];

    foreach ($ex->getViolations() as $violation) {
        $errors[] = $violation->getErrorMessage();
    }

    // Do something with the errors...
}
```

If you're using one of the `IValidator::tryValidate*()` methods, you can grab the violations from the violations array parameter:

```php
$violations = [];

if (!$validator->tryValidateObject($blogPost, $violations)) {
    $errors = [];

    foreach ($violations as $violation) {
        $errors[] = $violation->getErrorMessage();
    }

    // Do something with the errors...
}
```

<h3 id="error-message-templates">Error Message Templates</h3>

Aphiria allows you to configure how your error message IDs map to error message templates via error template registries.  The IDs are typically used in one of two ways:

1. As the error message template itself, which works best if you're not doing i18n
   - `DefaultErrorMessageTemplateRegistry` is recommended
2. As a sort of pointer (eg a slug) to the message template to use, which works best if you need to support i18n
   - Implementing your own template registry is recommended

Let's look at an example of option 2.  Let's say that your templates are stored in a PHP file and are separated by locale, eg:

```php
// These messages messages are in the ICU format
return [
    'en' => [
        'tooLong' => 'Value can not exceed {maxLength, plural, one {# character}, other {# characters}}'
    ],
    'es' => [
        'tooLong' => 'El valor no puede superar {maxLength, plural, one {un # caracter}, other {los # caracteres}}'
    ]
];
```

Let's create a registry to read from this file:

```php
use Aphiria\Validation\ErrorMessages\IErrorMessageTemplateRegistry;

final class ResourceFileErrorMessageTemplateRegistry implements IErrorMessageTemplateRegistry
{
    private array $errorMessages;
    private string $defaultLocale;

    public function __construct(string $path, private string $defaultLocale)
    {
        $this->errorMessages = require $path;
    }

    public function getErrorMessageTemplate(string $errorMessageId, string $locale = null): string
    {
        return $this->errorMessages[$locale][$errorMessageId] ?? $this->errorMessages[$this->defaultLocale][$errorMessageId];
    }
}
```

To use this registry, just pass it into your interpolator, and pass the interpolator into your validator.

```php
use Aphiria\Validation\ErrorMessages\IcuFormatErrorMessageInterpolator;
use Aphiria\Validation\Validator;

$errorMessageTemplates = new ResourceFileErrorMessageTemplateRegistry('/resources/errorMessageTemplates.php');
$errorMessageInterpolator = new IcuFormatErrorMessageInterpolator($errorMessageTemplates);

// Assume we've configured the object constraints
$validator = new Validator($objectConstraints, $errorMessageInterpolator);
```

You can override the default error message ID of a constraint by passing one in via the constructor:

```php
use Aphiria\Validation\Builders\ObjectConstraintsRegistryBuilder;
use Aphiria\Validation\Constraints\EmailConstraint;

$constraintsBuilder = new ObjectConstraintsRegistryBuilder();
$constraintsBuilder->class(User::class)
    ->hasPropertyConstraints('email', new EmailConstraint('email-invalid'));
```

<h3 id="built-in-error-message-interpolators">Built-In Error Message Interpolators</h3>

Aphiria comes with a couple error message interpolators.  `StringReplaceErrorMessageInterpolator` simply replaces `{placeholder}` in the constraints' error message templates with the constraints' placeholders.  It is the default interpolator, and is most suitable for applications that do not require i18n.

If you do require i18n and are using the <a href="http://userguide.icu-project.org/formatparse/messages" target="_blank">ICU format</a>, `IcuErrorMessageInterpolator` is probably the better choice.

<h2 id="validation-attributes">Validation Attributes</h2>

Aphiria offers the option to use attributes to map object properties and methods to constraints.  The benefit to doing this is that it keeps the validation rules close (literally) to your models.  Let's recreate the example in the [introduction](#introduction).

```php
use Aphiria\Validation\Constraints\Attributes\{Email, Required};

final class User
{
    public int $id;
    #[Email]
    public string $email;
    #[Required]
    public string $name;

    public function __construct(int $id, string $email, string $name)
    {
        $this->id = $id;
        $this->email = $email;
        $this->name = $name;
    }
}
```

Once you [configure your application to use attributes](#using-attributes), you can validate your objects [just like](#validating-objects) you do when not using attributes.

<h3 id="built-in-attributes">Built-In Attributes</h3>

The following attributes come with Aphiria:

* `Alpha`
* `Alphanumeric`
* `Between`
* `Date`
* `Each`
* `Email`
* `Equals`
* `In`
* `Integer`
* `IPAddress`
* `Max`
* `Min`
* `NotIn`
* `Numeric`
* `Regex`
* `Required`

<h3 id="using-attributes">Using Attributes</h3>

Before you can use attributes, you'll need to configure Aphiria to scan for them.  If you're using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, you can do so in `App`:

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Application\IModule;
use Aphiria\Framework\Application\AphiriaComponents;

final class App implements IModule
{
    use AphiriaComponents;

    public function build(IApplicationBuilder $appBuilder): void
    {
        $this->withValidatorAttributes($appBuilder);
    }
}
```

Otherwise, you can manually scan for attributes:

```php
use Aphiria\Validation\Constraints\Attributes\AttributeObjectConstraintsRegistrant;
use Aphiria\Validation\Constraints\Caching\FileObjectConstraintsRegistryCache;
use Aphiria\Validation\Constraints\ObjectConstraintsRegistrantCollection;
use Aphiria\Validation\Constraints\ObjectConstraintsRegistry;
use Aphiria\Validation\Validator;


// It's best to cache the results of scanning for attributes in production
if (\getenv('APP_ENV') === 'production') {  
    $constraintCache = new FileObjectConstraintsRegistryCache('/tmp/constraints.txt');
} else {
    $constraintCache = null;
}

$objectConstraints = new ObjectConstraintsRegistry();
$objectConstraintsRegistrants = new ObjectConstraintsRegistrantCollection($constraintCache);
$objectConstraintsRegistrants->add(new AttributeObjectConstraintsRegistrant(['PATH_TO_SCAN']));
$objectConstraintsRegistrants->registerConstraints($objectConstraints);

$validator = new Validator($objectConstraints);
```

<h2 id="validating-request-bodies">Validating Request Bodies</h2>

It's possible to use the validation library along with <a href="https://symfony.com/doc/current/components/serializer.html" target="_blank">Symfony's serialization component</a> to validate deserialized request bodies.  Read the [controller documentation](controllers.md#validating-request-bodies) for more details.

<h1 id="doc-title">Validation</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Introduction](#introduction)
   1. [Creating A Validator](#creating-a-validator)
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
5. [Validating Request Bodies](#validating-request-bodies)

</div>

</nav>

<h2 id="introduction">Introduction</h2>

Validating your data, especially input, is critical for ensuring that your application runs smoothly.  Let's take a look at how you can do this with your POPOs in Aphiria.  Assume we have the following model in your application:

```php
use Aphiria\Validation\Constraints\Attributes\{Email, Required};

final class User
{
    public int $id;
    #[Email]
    public string $email;
    #[Required]
    public string $name;
}
```

Once you've [set up your validator](#creating-a-validator), validating a `User` instance is as simple as:

```php
$user = new User(123, 'dave@example.com', 'Dave');
$validator->validateObject($user);
```

If the object was not valid, a `ValidationException` will be thrown.  That's it - validation, made simple.

<h3 id="creating-a-validator">Creating A Validator</h3>

If you're using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, an instance of `IValidator` will already be [bound](dependency-injection.md#binders) to the DI container, which you can [inject](dependency-injection.md).

You can enable attributes in `GlobalModule`:

```php
use Aphiria\Application\Builders\IApplicationBuilder;
use Aphiria\Framework\Application\AphiriaModule;

final class GlobalModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
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

If you prefer to not use attributes, you can use a fluent syntax to manually register constraints instead:

```php
use Aphiria\Validation\Builders\ObjectConstraintsRegistryBuilder;
use Aphiria\Validation\Constraints\EmailConstraint;
use Aphiria\Validation\Constraints\RequiredConstraint;
use Aphiria\Validation\Validator;

// Set up our validator
$constraintsBuilder = new ObjectConstraintsRegistryBuilder();
$constraintsBuilder->class(User::class)
    ->hasPropertyConstraints('email', new EmailConstraint())
    ->hasPropertyConstraints('name', new RequiredConstraint());
$validator = new Validator($constraintsBuilder->build());
```

> **Note:** The best place to [manually register constraints](configuration.md#component-validator) on your classes is in a [module](configuration.md#modules).

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

Name | Attribute | Description
------ | ------ | ------
`AlphaConstraint` | `Alpha` | The value must only contain alphabet characters
`AlphanumericConstraint` | `Alphanumeric` | The value must only contain alphanumeric characters
`BetweenConstraint` | `Between` | The value must fall in between two values (takes in whether or not the min and max are inclusive)
`CallbackConstraint` | N/A | The value must satisfy a callback that returns a boolean
`DateConstraint` | `Date` | The value must match a date-time format
`EachConstraint` | `Each` | The value must satisfy a list of constraints (takes in a list of `IConstraint`)
`EqualsConstraint` | `Equals` | The value must equal a value
`InConstraint` | `In` | The value must be in a list of acceptable values
`IntegerConstraint` | `Integer` | The value must be an integer
`IPAddressConstraint` | `IPAddress` | The value must be an IP address
`MaxConstraint` | `Max` | The value cannot exceed a max value (takes in whether or not the max is inclusive)
`MinConstraint` | `Min` | The value cannot go below a min value (takes in whether or not the min is inclusive)
`NotInConstraint` | `NotIn` | The value must not be in a list of values
`NumericConstraint` | `Numeric` | The value must be numeric
`RegexConstraint` | `Regex` | The value must satisfy a regular expression
`RequiredConstraint` | `Required` | The value must not be null

<h3 id="custom-constraints">Custom Constraints</h3>

Let's say you want a custom constraint that enforces the max length of a string.  Easy - just implement `IConstraint`.

```php
use Aphiria\Validation\Constraints\IConstraint;

final class MaxLengthConstraint implements IConstraint
{
    public function __construct(private int $maxLength) {}

    public function getErrorMessageId(): string
    {
        return 'Length cannot exceed {maxLength}';
    }

    public function getErrorMessagePlaceholders(mixed $value): array
    {
        return ['maxLength' => $this->maxLength];
    }

    public function passes(mixed $value): bool
    {
        if (!\is_string($value)) {
            throw new \InvalidArgumentException('Value must be string');
        }

        return \mb_strlen($value) <= $this->maxLength;
    }
}
```

Let's set up an attribute for this constraint.

```php
use Aphiria\Validation\Constraints\Attributes\ConstraintAttribute;

final class MaxLength extends ConstraintAttribute
{
    public function __construct(public int $maxLength, string $errorMessageId = null)
    {
        parent::__construct($errorMessageId);
    }
    
    public function createConstraintFromAttribute(): MaxLengthConstraint
    {
        return new MaxLengthConstraint($this->maxLength);
    }
}
```

You can now use this constraint just like any other built-in constraint:

```php
final class BlogPost
{
    #[MaxLength(32)]
    public string $title;
    #[MaxLength(1000)]
    public string $content;
}
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
        $errors[] = $violation->errorMessage;
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
        $errors[] = $violation->errorMessage;
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
// These messages are in the ICU format
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

    public function __construct(string $path, private string $defaultLocale)
    {
        $this->errorMessages = require $path;
    }

    public function getErrorMessageTemplate(string $errorMessageId, string $locale = null): string
    {
        return $this->errorMessages[$locale][$errorMessageId] 
            ?? $this->errorMessages[$this->defaultLocale][$errorMessageId];
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

You can override the default error message ID of a constraint:

```php
use Aphiria\Validation\Constraints\Attributes\Required;

final class BlogPost
{
    #[Required('title-invalid')]
    public string $title;
    #[Required('content-invalid')]
    public string $content;
}
```

<h3 id="built-in-error-message-interpolators">Built-In Error Message Interpolators</h3>

Aphiria comes with a couple error message interpolators.  `StringReplaceErrorMessageInterpolator` simply replaces `{placeholder}` in the constraints' error message templates with the constraints' placeholders.  It is the default interpolator, and is most suitable for applications that do not require i18n.

If you do require i18n and are using the <a href="http://userguide.icu-project.org/formatparse/messages" target="_blank">ICU format</a>, `IcuErrorMessageInterpolator` is probably the better choice.

<h2 id="validating-request-bodies">Validating Request Bodies</h2>

It's possible to use the validation library to validate deserialized request bodies.  Read the [controller documentation](controllers.md#validating-request-bodies) for more details.

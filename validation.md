<h1 id="doc-title">Validation</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Introduction](#introduction)
2. [Validating Data](#validating-objects)
   1. [Validating Objects](#validating-objects)
   2. [Validating Properties](#validating-properties)
   3. [Validating Methods](#validating-methods)
   4. [Validating Values](#validating-values)
3. [Constraints](#constraints)
   1. [Built-In Constraints](#built-in-constraints)
   2. [Custom Constraints](#custom-constraints)
4. [Error Messages](#error-messages)
   1. [Built-In Error Message Compilers](#built-in-error-message-compilers)
5. [Validation Annotations](#validation-annotations)
   1. [Built-In Annotations](#built-in-annotations)
   2. [Using Annotations](#using-annotations)

</div>

</nav>

<h2 id="introduction">Introduction</h2>

Validating your data, especially input, is critical for ensuring that your application runs smoothly.  Let's take a look at how you can do this with your POPOs in Aphiria.  Assume we have the following model in your application:

```php
final class User
{
    public int $id;
    public string $email;
    public string $name;

    public function __construct(int $id, string $email, string $name)
    {
        $this->id = $id;
        $this->email = $email;
        $this->name = $name;
    }
}
```

Let's set up some constraints.

```php
use Aphiria\Validation\Builders\ObjectConstraintsRegistryBuilder;
use Aphiria\Validation\Constraints\{EmailConstraint, Requiredconstraint};
use Aphiria\Validation\ValidationContext;
use Aphiria\Validation\Validator;
use App\Users\User;

// Set up our validator
$constraintsBuilder = new ObjectConstraintsRegistryBuilder();
$constraintsBuilder->class(User::class)
    ->hasPropertyConstraints('email', new EmailConstraint())
    ->hasPropertyConstraints('name', new Requiredconstraint());
$validator = new Validator($constraintsBuilder->build());

// Let's validate
$user = new User(123, 'dave@example.com', 'Dave');
$validator->validateObject($user, new ValidationContext($user));
```

If the object was not valid, a `ValidationException` will be thrown.  That's it - validation, made simple.

<h2 id="validating-data">Validating Data</h2>

To validate data, you need three things:

1. The data itself
2. A list of [constraints](#constraints) to apply
3. A context in which to perform the validation

Having a context allows us to keep track of things like circular dependencies, which property/method we're validating, and any violations found during validation.

<h3 id="validating-objects">Validating Objects</h3>

To validate an object, simply map the properties and methods in that object to constraints.  Aphiria will then recursively validate the object and any properties/methods that contain objects.  Use `ObjectConstraintsRegistryBuilder` to help set up the constraints on your objects' properties/methods [like in the above example](#introduction).  To validate an object, we have two options:

```php
$blogPost = new BlogPost('How to Reticulate Splines');

// Will throw a ValidationException if $blogPost is invalid
$valdiator->validateObject($blogPost, new ValidationContext($blogPost));

// Or

// Will return true if $blogPost is valid, otherwise false
if ($validator->tryValidateObject($blogPost, new ValidationContext($blogPost))) {
    // ...
}
```

<h3 id="validating-properties">Validating Properties</h3>

You can validate an individual property from an object:

```php
$blogPost = new BlogPost('How to Reticulate Splines');

// Will throw a ValidationException if $blogPost->title is invalid
$valdiator->validateProperty($blogPost, 'title', new ValidationContext($blogPost, 'title'));

// Or

// Will return true if $blogPost->title is valid, otherwise false
if ($validator->tryValidateProperty($blogPost, 'title', new ValidationContext($blogPost, 'title'))) {
    // ...
}
```

In the case that the property holds an object value, it will be recursively validated, too.

<h3 id="validating-methods">Validating Methods</h3>

You can validate an individual method very similarly to how you validate properties:

```php
$blogPost = new BlogPost('How to Reticulate Splines');

// Will throw a ValidationException if $blogPost->getTitleSlug() is invalid
$valdiator->validateMethod($blogPost, 'getTitleSlug', new ValidationContext($blogPost, null, 'getTitleSlug'));

// Or

// Will return true if $blogPost->getTitleSlug() is valid, otherwise false
if ($validator->tryValidateMethod($blogPost, 'getTitleSlug', new ValidationContext($blogPost, null, 'getTitleSlug'))) {
    // ...
}
```

In the case that the method holds an object value, it will also be recursively validated.

<h3 id="validating-values">Validating Values</h3>

If you want to validate an individual value, you can:

```php
// Will throw a ValidationException if $email is invalid
$valdiator->validateValue($email, [new EmailConstraint()], new ValidationContext($email));

// Or

// Will return true if $email is valid, otherwise false
if ($validator->tryValidateValue($email, [new EmailConstraint()], new ValidationContext($email))) {
    // ...
}
```

<h2 id="constraints">Constraints</h2>

A constraint is something that a value must pass to be considered valid.  For example, if you want a value to only contain alphabet characters, you can enforce the `AlphaConstraint` on it.  All constraints must implement `IConstraint`.

<h3 id="built-in-constraints">Built-In Constraints</h3>

Aphiria comes with some useful constraints built-in:

* `AlphaConstraint`
* `AlphanumericConstraint`
* `BetweenConstraint`
  * Takes in a min value, a max value, whether or not the min is inclusive, and whether or not the max is inclusive
* `CallbackConstraint`
  * Takes in a callback that accepts a value and returns whether or not the value passes
* `DateConstraint`
  * Takes in single or multiple `DateTime` formats that can be used to match the input value
* `EachConstraint`
  * Takes in single or multiple `IConstraint`s and applies them to each value in the input value (input value must be iterable)
* `EmailConstraint`
* `EqualsConstraint`
  * Takes in a value that must be equal to the input value
* `InConstraint`
  * Takes in a list of values that must contain the input value
* `IntegerConstraint`
* `IPAddressConstraint`
* `MaxConstraint`
  * Takes in a max value and whether or not it is inclusive
* `MinConstraint`
  * Takes in a min value and whether or not it is inclusive
* `NotInConstraint`
  * Takes in a list of values that must not contain the input value
* `NumericConstraint`
* `RegexConstraint`
  * Takes in a regex that will be applied to determine if the input value passes
* `RequiredConstraint`

<h3 id="custom-constraints">Custom Constraints</h3>

Creating a custom constraint is simple - just implement `IConstraint`.

```php
use Aphiria\Validation\Constraints\IConstraint;
use Aphiria\Validation\ValidationContext;

final class MaxLengthConstraint implements IConstraint
{
    private int $maxLength;

    public function __construct(int $maxLength)
    {
        $this->maxLength = $maxLength;
    }

    public function getErrorMessageId(): string
    {
        return 'Length cannot exceed {maxLength}';
    }

    public function getErrorMessagePlaceholders(): array
    {
        return ['maxLength' => $this->maxLength];
    }

    public function passes($value, ValidationContext $context): bool
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

Error messages provide human-readable explanations of what failed during validation.  `IConstraint` provides an error message ID, which can be used in two ways:

1. As the error message itself, which works best if you're not doing i18n
2. As a sort of pointer (eg a slug) to the message to use, which works best if you're using a third-party i18n library

Constraints also provide error message placeholders, which can give more specifics on why a constraint failed.  For example, `MaxConstraint` has a default error message ID of `Field must be less than {max}`, and it provides a `max` error message placeholder so that you can display the actual max in the error message.

To actually compile error messages, you can use an instance of `IErrorMessageCompiler`.  Depending on how you're validating a value, there are different ways of grabbing the constraint violations.  If you're using `IValidator::validate*()` methods, you can grab the violations from the `ValidationException`:

```php
use Aphiria\Validation\ErrorMessages\StringReplaceErrorMessageCompiler;
use Aphiria\Validation\ValidationException;

$errorMessageCompiler = new StringReplaceErrorMessageCompiler();

try {
    $validator->validateObject($blogPost);
} catch (ValidationException $ex) {
    $compiledErrorMessages = [];

    foreach ($ex->getValidationContext()->getConstraintViolations() as $violation) {
        $failedConstraint = $violation->getConstraint();
        $compiledErrorMessages[] = $errorMessageCompiler->compile(
            $failedConstraint->getErrorMessageId(), 
            $failedConstraint->getErrorMessagePlaceholders()
        );
    }

    // Do something with the compiled error messages
}
```

If you're using one of the `IValidator::tryValidate*()` methods, you can grab the violations from the `ValidationContext`:

```php
use Aphiria\Validation\ValidationContext;

$validationContext = new ValidationContext($blogPost);

if (!$validator->tryValidateObject($blogPost, $validationContext)) {
    $compiledErrorMessages = [];
    
    foreach ($validationContext->getConstraintViolations() as $violation) {
        // Same as in the above example...
    }

    // Do something with the compiled error messages
}
```

<h3 id="built-in-error-message-compilers">Built-In Error Message Compilers</h3>

Aphiria comes with a couple error message compilers.  `StringReplaceErrorMessageCompiler` simply replaces `{placeholder}` in the constraints' error message IDs with the constraints' placeholders.  It is the default compiler, and is most suitable for applications that do not require i18n.

<h2 id="validation-annotations">Validation Annotations</h2>

Aphiria offers the option to use annotations to map object properties and methods to constraints.  The benefit to doing this is that it keeps the validation rules close (literally) to your models.  Let's recreate the example in the [introduction](#introduction).

```php
use Aphiria\Validation\Constraints\Annotations\{Email, Required};

final class User
{
    public int $id;
    /** @Email */
    public string $email;
    /** @Required */
    public string $name;

    public function __construct(int $id, string $email, string $name)
    {
        $this->id = $id;
        $this->email = $email;
        $this->name = $name;
    }
}
```

Once you [configure your application to use annotations](#using-annotations), you can validate your objects [just like](#validating-objects) you do when not using annotations.

<h3 id="built-in-annotations">Built-In Annotations</h3>

The following annotations come with Aphiria:

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

<h3 id="using-annotations">Using Annotations</h3>

To actually use annotations, you'll need to configure Aphiria to scan for them.  The [configuration](application-builders.md) library provides a convenience method for this:

```php
use Aphiria\Configuration\AphiriaComponentBuilder;
use Aphiria\Validation\Constraints\Annotations\AnnotationObjectConstraintsRegistrant;

// Assume we already have $container set up
$annotationConstraintRegistrant = new AnnotationObjectConstraintsRegistrant(['PATH_TO_SCAN']);
$container->bindInstance(AnnotationObjectConstraintsRegistrant::class, $annotationConstraintRegistrant);

(new AphiriaComponentBuilder($container))
    ->withValidationComponent($appBuilder)
    ->withValidationAnnotations($appBuilder);
```

If you're not using the configuration library, you can manually scan for annotations:

```php
use Aphiria\Validation\Constraints\Annotations\AnnotationObjectConstraintsRegistrant;
use Aphiria\Validation\Constraints\Caching\CachedObjectConstraintsRegistrant;
use Aphiria\Validation\Constraints\Caching\FileObjectConstraintsRegistryCache;
use Aphiria\Validation\Constraints\ObjectConstraintsRegistry;
use Aphiria\Validation\Validator;

$objectConstraints = new ObjectConstraintsRegistry();
$annotationConstraintRegistrant = new AnnotationObjectConstraintsRegistrant(['PATH_TO_SCAN']);

// It's best to cache the results of scanning for annotations in production
if (\getenv('APP_ENV') === 'production') {  
    $constraintCache = new FileObjectConstraintsRegistryCache('/tmp/constraints.txt');
    $constraintRegistrant = new CachedObjectConstraintsRegistrant($constraintCache);
    $constraintRegistrant->addConstraintRegistrant($annotationConstraintRegistrant);
    $constraintRegistrant->registerConstraints($objectConstraints);
} else {
    $annotationConstraintRegistrant->registerConstraints($objectConstraints);
}

$validator = new Validator($objectConstraints);
```

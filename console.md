<h1 id="doc-title">Console</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
   1. [Why Is This Library Included?](#why-is-this-library-included)
2. [Running Commands](#running-commands)
   1. [Getting Help](#getting-help)
3. [Creating Commands](#creating-commands)
   1. [Input](#input)
   2. [Arguments](#arguments)
   3. [Options](#options)
   4. [Output](#output)
   5. [Manually Registering Commands](#manually-registering-commands)
   6. [Calling From Code](#calling-from-code)
4. [Command Attributes](#command-attributes)
    1. [Example](#command-attribute-example)
    4. [Scanning For Attributes](#scanning-for-attributes)
5. [Prompts](#prompts)
   1. [Confirmation](#confirmation)
   2. [Multiple Choice](#multiple-choice)
   3. [Hiding Input](#hiding-input)
6. [Formatters](#formatters)
   1. [Padding](#padding)
   2. [Tables](#tables)
   3. [Progress Bars](#progress-bars)
7. [Style Elements](#style-elements)
   1. [Built-In Elements](#built-in-elements)
   2. [Custom Elements](#custom-elements)
   3. [Overriding Built-In Elements](#overriding-built-in-elements)
8. [Built-In Commands](#built-in-commands)

</div>

</nav>
  
<h2 id="basics">Basics</h2>

Console applications are great for administrative tasks and code generation.  With Aphiria, you can easily create your own console commands, display question prompts, and use HTML-like syntax for output styling.

If you're already using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, you can skip to the [next section](#running-commands).  Otherwise, let's create a file called _aphiria_ in your project's root directory and paste the following code into it:

```php
#!/usr/bin/env php
<?php

use Aphiria\Console\Commands\CommandRegistry;
use Aphiria\Console\ConsoleGateway;
use Aphiria\Console\Input\Compilers\InputCompiler;
use Aphiria\Console\StatusCode;
use Aphiria\DependencyInjection\Container;
use Aphiria\Framework\Console\ConsoleApplication;

require __DIR__ . '/vendor/autoload.php';

$commands = new CommandRegistry();

// Register your commands here...

$gateway = new ConsoleGateway($commands, new Container());
$input = new InputCompiler($commands)->compile($_SERVER['argv'] ?? []);
$app = new ConsoleApplication($gateway, $input);
$status = $app->run();

exit(\is_int($status) ? $status : $status instance StatusCode ? $status->value : StatusCode::Ok->value);
```

Now, you're set to start [running commands](#running-commands).

<h3 id="why-is-this-library-included">Why Is This Library Included?</h3>

At first glance, including a console library in an API framework might seem weird.  However, there are some tasks, [such as clearing framework caches](#built-in-commands), that are most easily accomplished with console commands.  We decided not to use another console library because we felt we could provide a better developer experience than most, eg by providing [attribute support](#command-attributes) and a great [fluent syntax](configuration.md#component-console-commands) for configuring commands.

<h2 id="running-commands">Running Commands</h2>

To run commands, type `php aphiria COMMAND_NAME` into a terminal from the directory that Aphiria is installed in.

<h3 id="getting-help">Getting Help</h3>

To get help with any command, use the help command:

```bash
php aphiria help COMMAND_NAME
```

<h2 id="creating-commands">Creating Commands</h2>

In Aphiria, a command defines the name, [arguments](#arguments), and [options](#options) that make up a command.  Each command has a command handler with method `handle()`, which is what actually processes a command. Command handlers can return a `StatusCode` enum, an integer (useful for custom status codes), or nothing, which implies a successful status code.

Let's take a look at an example:

```php

use Aphiria\Console\Commands\Attributes\{Argument, Command, Option};
use Aphiria\Console\Commands\ICommandHandler;
use Aphiria\Console\Input\{ArgumentType, Input, OptionType};
use Aphiria\Console\Output\IOutput;

#[
    Command('greet', description: 'Greets a person'),
    Argument('name', type: ArgumentType::Required, description: 'The name to greet'),
    Option('yell', type: OptionType::OptionalValue, shortName: 'y', description: 'Yell the greeting?', defaultValue: 'yes')
]
final class GreetingCommandHandler implements ICommandHandler
{
    public function handle(Input $input, IOutput $output)
    {
        $greeting = "Hello, {$input->arguments['name']}";
        
        if ($input->options['yell'] === 'yes') {
            $greeting = \strtoupper($greeting);
        }
    
        $output->writeln($greeting);
    }
}
```

<h3 id="input">Input</h3>

The following properties are available to you in `Input`:

```php
$input->commandName; // The name of the command that was invoked
$input->arguments['argName']; // The value of "argName"
$input->options['optionName']; // The value of "optionName"
```

If you're checking to see if an option that does not have a value is set, use `array_key_exists('optionName', $input->options)` - the value will be `null`, and `isset()` will return `false`.

> **Note:** `$input->options` stores option values by their long names.  Do not try to access them by their short names.

<h3 id="arguments">Arguments</h3>

Console commands can accept arguments from the user.  Arguments can be required, optional, and/or arrays.  Array arguments allow a variable number of arguments to be passed in, like `php aphiria foo arg1 arg2 arg3 ...`.  The only catch is that array arguments must be the last argument defined for the command.  If you need to specify that an argument can be multiple types, eg required and an array, just pass in an array of types.

```php
use Aphiria\Console\Input\Argument;
use Aphiria\Console\Input\ArgumentType;

// The argument will be required and an array
$type = [ArgumentType::Required, ArgumentType::IsArray];
// The description argument is used by the help command
$argument = new Argument('foo', $type, 'The foo argument');
```

>**Note:** Like array arguments, optional arguments must appear after any required arguments.

<h3 id="options">Options</h3>

You might want different behavior in your command depending on whether or not an option is set.  This is possible using `Option`.  Options have two formats:

1. [Short](#short-names), eg "-h"
2. [Long](#long-names), eg "--help"

<h4 id="short-names">Short Names</h4>

Short option names are always a single letter.  Multiple short options can be grouped together.  For example, `-rf` means that options with short codes "r" and "f" have been specified.  The default value will be used for short options.

<h4 id="long-names">Long Names</h4>

Long option names can specify values in two ways:  `--foo=bar` or `--foo bar`.  If you only specify `--foo` for an optional-value option, then the default value will be used.

<h4 id="array-options">Array Options</h4>

Options can be arrays, eg `--foo=bar --foo=baz` will set the "foo" option to `["bar", "baz"]`.

Like arguments, multiple option types can be specified with an array of types.

```php
use Aphiria\Console\Input\Option;
use Aphiria\Console\Input\OptionType;

$type = [OptionType::IsArray, OptionType::RequiredValue];
$option = new Option('foo', $type, 'f', 'The foo option');
```

<h3 id="output">Output</h3>

Outputs allow you to write messages to an end user.  The different outputs include:

1. `Aphiria\Console\Output\ConsoleOutput`
   * Writes output to the console, and is the default output
2. `Aphiria\Console\Output\SilentOutput`
   * Used when we don't want any messages to be written

Each output offers a few methods:

1. `readLine()`
   * Reads a line of input
2. `write()`
   * Writes a message to the existing line
3. `writeln()`
   * Writes a message to a new line
4. `clear()`
   * Clears the current screen
   * Only works in `ConsoleOutput`

<h3 id="manually-registering-commands">Manually Registering Commands</h3>

If you're using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, this is already handled for you, and you can skip this section.  Otherwise, you'll have to register commands so that your application knows about them.  If you're using attributes, read [this section](#scanning-for-attributes) to learn how to manually register attribute commands.  If you're using the application builder library, refer to [its documentation](configuration.md#component-console-commands) to learn how to manually register your commands to your app.

Let's manually register a command to the application:

```php
use Aphiria\Console\Commands\Command;
use Aphiria\Console\Commands\CommandRegistry;

$commands = new CommandRegistry();
$greetingCommand = new Command('greet', arguments: [/* ... */], options: [/* ... */]);
$commands->registerCommand($greetingCommand, GreetingCommandHandler::class);

// Run the application (see above how to do this)...
```

To call this command, run this from the command line:

```bash
php aphiria greet Dave -y
```

This will output:

```
HELLO, DAVE
```

> **Note:**  Command handlers will only be resolved when they're called, which is especially useful when your handler is a class with expensive-to-instantiate dependencies, such as database connections.

<h3 id="calling-from-code">Calling From Code</h3>

It's possible to call a command from another command by injecting `ICommandHandler` into your command handler, which will be bound to an instance of `ConsoleGateway`.  Here, we'll call the `bar` command with an argument and option:

```php
use Aphiria\Console\Commands\Attributes\Command;
use Aphiria\Console\Commands\ICommandHandler;
use Aphiria\Console\Input\Input;
use Aphiria\Console\Output\IOutput;

#[Command('foo')]
final class FooCommandHandler implements ICommandHandler
{
    public function __construct(private ICommandHandler $gateway) {}

    public function handle(Input $input, IOutput $output)
    {
        $this->gateway->handle(new Input('bar', ['arg1' => null], ['option1' => 'value']), $output);
    }
}
```

If you want to call the other command but not write its output, use the `SilentOutput` output.

> **Note:** If a command is being called by a lot of other commands, it might be best to refactor its actions into a separate class.  This way, it can be used by multiple commands without the extra overhead of calling console commands through PHP code.

<h2 id="command-attributes">Command Attributes</h2>

It's convenient to define your command alongside your command handler so you don't have to jump back and forth remembering what arguments or options your command takes.  Aphiria offers the option to do so via attributes.

<h3 id="command-attribute-example">Command Attribute Example</h3>

Let's look at an example that duplicates the [greeting example from above](#registering-commands):

```php
use Aphiria\Console\Commands\Attributes\{Argument, Command, Option};
use Aphiria\Console\Input\{ArgumentType, OptionType};

 #[
    Command('greet', 'Greets a person'),
    Argument('name', ArgumentType::Required, 'The name to greet'),
    Option('yell', OptionType::OptionalValue, 'y', 'Yell the greeting', 'yes')
 ]
final class GreetingCommandHandler implements ICommandHandler
{
    public function handle(Input $input, IOutput $output)
    {
        // ...
    }
}
```

<h3 id="scanning-for-attributes">Scanning For Attributes</h3>

Before you can use attributes, you'll need to configure Aphiria to scan for them.  If you're using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, you can do so in `GlobalModule`:

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Framework\Application\AphiriaModule;

final class GlobalModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        $this->withCommandAttributes($appBuilder);
    }
}
```

Otherwise, you can manually configure your app to scan for attributes:

```php
use Aphiria\Console\Commands\Attributes\AttributeCommandRegistrant;
use Aphiria\Console\Commands\CommandRegistry;

// Assume we already have $container set up
$commands = new CommandRegistry();
$attributeCommandRegistrant = new AttributeCommandRegistrant(['PATH_TO_SCAN'], $container);
$attributeCommandRegistrant->registerCommands($commands);
```

<h2 id="prompts">Prompts</h2>

Prompts are great for asking users for input beyond what is accepted by arguments.  For example, you might want to confirm with a user before doing an administrative task, or you might ask her to select from a list of possible choices.

<h3 id="confirmation">Confirmation</h3>

To ask a user to confirm an action with a simple "y" or "yes", use a confirmation prompt.

```php
use Aphiria\Console\Output\Prompts\Confirmation;
use Aphiria\Console\Output\Prompts\Prompt;

$prompt = new Prompt();
// This will return true if the answer began with "y" or "Y"
$prompt->ask(new Confirmation('Are you sure you want to continue?'), $output);
```

<h3 id="multiple-choice">Multiple Choice</h3>

Multiple choice questions are great for listing choices that might otherwise be difficult for a user to remember.

```php
use Aphiria\Console\Output\Prompts\MultipleChoice;

$choices = ['Boeing 747', 'Boeing 757', 'Boeing 787'];
$question = new MultipleChoice('Select your favorite airplane', $choices);
$prompt->ask($question, $output);
```

This will display:

```php
Select your favorite airplane
  1) Boeing 747
  2) Boeing 757
  3) Boeing 787
  >
```

If the `$choices` array is associative, then the keys will map to values rather than 1)...N).

<h3 id="hiding-input">Hiding Input</h3>

For security reasons, such as when entering a password, you might want to hide a user's input as they're typing it.  To do so, just mark a question as hidden:

```php
$question = new Question('Password', isHidden: true);
$prompt->ask($question, $output);
```

<h2 id="formatters">Formatters</h2>

Formatting your output helps make it more readable.  Aphiria provides a few common formatters out of the box.

<h3 id="padding">Padding</h3>

The `PaddingFormatter` formatter allows you to create column-like output.  It accepts an array of column values.  The second parameter is a callback that will format each row's contents.  Let's look at an example:

```php
use Aphiria\Console\Output\Formatters\PaddingFormatter;

$paddingFormatter = new PaddingFormatter();
$rows = [
    ['George', 'Carlin', 'great'],
    ['Chris', 'Rock', 'good'],
    ['Jim', 'Gaffigan', 'pale']
];
$paddingFormatter->format($rows, fn ($row) => $row[0] . ' - ' . $row[1] . ' - ' . $row[2]);
```

This will return:
```
George - Carlin   - great
Chris  - Rock     - good
Jim    - Gaffigan - pale
```

You can pass in an options parameter to customize the output of the padding formatter:

```php
use Aphiria\Console\Output\Formatters\PaddingFormatterOptions;

$options = new PaddingFormatterOptions(
    paddingString: ' ',
    padAfter: true,
    eolChar: "\n"
);
$paddingFormatter->format($rows, fn ($row) => $row[0] . ' - ' . $row[1] . ' - ' . $row[2], $options);
```

> **Note:** You can set a default set of options for `PaddingFormatter` and `TableFormatter` in their constructors if you do not want to pass in options on every call to `format()`.

<h3 id="tables">Tables</h3>

ASCII tables are a great way to show tabular data in a console.  To create a table, use `TableFormatter`:

```php
use Aphiria\Console\Output\Formatters\TableFormatter;

$table = new TableFormatter();
$rows = [
    ['Sean', 'Connery'],
    ['Pierce', 'Brosnan']
];
$table->format($rows);
```

This will return:

```
+--------+---------+
| Sean   | Connery |
| Pierce | Brosnan |
+--------+---------+
```

Headers can also be included in tables:

```php
$headers = ['First', 'Last'];
$table->format($rows, $headers);
```

This will return:

```
+--------+---------+
| First  | Last    |
+--------+---------+
| Sean   | Connery |
| Pierce | Brosnan |
+--------+---------+
```

Like `PaddingFormatter`, you can specify some options to customize the look of tables:

```php
use Aphiria\Console\Output\Formatters\TableFormatterOptions;

$options = new TableFormatterOptions(
    cellPaddingString: ' ',
    horizontalBorderChar: '-',
    verticalBorderChar: '|',
    intersectionChar: '+',
    padAfter: true,
    eolChar: "\n"
);
$table->format($rows, $headers, $options);
```
    
<h3 id="progress-bars">Progress Bars</h3>

Progress bars help visually indicate to a user the progress of a long-running task, like this one:

```
[=================50%--------------------] 50/100
Time remaining: 15 secs
```  

Creating one is simple - you just create a `ProgressBar`, and specify the formatter to use:

```php
use Aphiria\Console\Output\Formatters\{ProgressBar, ProgressBarFormatter};

// Assume our output is already created
$formatter = new ProgressBarFormatter($output);
// Set the maximum number of "steps" in the task to 100
$progressBar = new ProgressBar(100, $formatter);
```

You can advance your progress bar:

```php
$progressBar->advance();

// Or with a custom step

$progressBar->advance(2);
```

Alternatively, you can set a specific progress:

```php
$progressBar->progress = 50;
```

To explicitly complete the progress bar, call

```php
$progressBar->complete();
```

Each time progress is made, the formatter will be update.

<h4 id="customizing-progress-bars">Customizing Progress Bars</h4>

You may customize the look of your progress bar with some options:

```php
use Aphiria\Console\Output\Formatters\ProgressBarFormatterOptions;

$options = new ProgressBarFormatterOptions(
    progressBarWidth: 100,
    outputFormat: '%bar% - Time remaining: %timeRemaining%',
    completedProgressChar: '=',
    remainingProgressChar: '-',
    redrawFrequency: 1
);
$progressBar = new ProgressBar(100, $formatter, $options);
```

If you'd like to customize the format of the progress bar text, you may by specifying `sprintf()`-encoded text.  The following placeholders are built in for you to use in the `$outputFormat` parameter of `ProgressBarFormatterOptions`:

* `%progress%` - The current progress
* `%maxSteps%` - The max number of steps
* `%bar%` - The actual progress bar that's drawn
* `%timeRemaining%` - The amount of time remaining
* `%percent%` - The current progress as a percentage

<h2 id="style-elements">Style Elements</h2>

Aphiria supports HTML-like style elements to perform basic output formatting like background color, foreground color, boldening, and underlining.  For example, writing:

```
<b>Hello!</b>
```

...will output "<b>Hello!</b>".  You can even nest elements:

```
<u>Hello, <b>Dave</b></u>
```

..., which will output an underlined string where "Dave" is both bold AND underlined.

<h3 id="built-in-elements">Built-In Elements</h3>

The following elements come built-into Aphiria:

* &lt;success&gt;&lt;/success&gt;
* &lt;info&gt;&lt;/info&gt;
* &lt;question&gt;&lt;/question&gt;
* &lt;comment&gt;&lt;/comment&gt;
* &lt;error&gt;&lt;/error&gt;
* &lt;fatal&gt;&lt;/fatal&gt;
* &lt;b&gt;&lt;/b&gt;
* &lt;u&gt;&lt;/u&gt;

<h3 id="custom-elements">Custom Elements</h3>

You can create your own style elements.  Elements are registered to `ElementRegistry`.

```php
use Aphiria\Console\Commands\CommandRegistry;
use Aphiria\Console\Output\Compilers\Elements\{Colors, Element, ElementRegistry, Style, TextStyles};
use Aphiria\Console\Output\Compilers\OutputCompiler;
use Aphiria\Console\Output\ConsoleOutput;
use Aphiria\DependencyInjection\Container;
use Aphiria\Framework\Console\ConsoleApplication;

$commands = new CommandRegistry();

// Register your commands here...

// Register a custom element
$elements = new ElementRegistry();
$elements->registerElement(
    new Element('foo', new Style(Colors::BLACK, Colors::YELLOW, [TextStyles::BOLD])
);
$outputCompiler = new OutputCompiler($elements);
$output = new ConsoleOutput($outputCompiler);

// Now, pass it into the app
$gateway = new ConsoleGateway($commands, new Container());
$input = new InputCompiler($commands)->compile($_SERVER['argv'] ?? []);
$app = new ConsoleApplication($gateway, $input, $output);
$status = $app->run();

exit(\is_int($status) ? $status : $status instance StatusCode ? $status->value : StatusCode::Ok->value);
```

<h3 id="overriding-built-in-elements">Overriding Built-In Elements</h3>

To override a built-in element, just re-register it:

```php
$elements->registerElement(
    new Element('success', new Style(Colors::GREEN, Colors::BLACK))
);
```

<h2 id="built-in-commands">Built-In Commands</h2>

Aphiria provides some commands out of the box to make it easier to work with the framework.

Name | Description
------ | ------
`app:serve` | Runs your application locally
`framework:flushcaches` | Flushes all the framework's caches, eg the binder metadata, constraints, command, route, and trie caches
`route:list` | Lists all the routes in your application

If you're using the <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a>, you can register all framework commands in `GlobalModule`:

```php
use Aphiria\Application\IApplicationBuilder;
use Aphiria\Framework\Application\AphiriaModule;

final class GlobalModule extends AphiriaModule
{
    public function configure(IApplicationBuilder $appBuilder): void
    {
        $this->withFrameworkCommands($appBuilder);
    }
}
```

To exclude some built-in commands so that you can override them with your own implementation, eg `app:serve`, pass in an array of command names.

```php
$this->withFrameworkCommands($appBuilder, ['app:serve']);
```

<h1 id="doc-title">Configuration</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Basics](#basics)
   1. [Reading PHP Files](#reading-php-files)
   2. [Reading JSON Files](#reading-json-files)
   3. [Custom File Readers](#custom-file-readers)
2. [Global Configuration](#global-configuration)
   1. [Building The Global Configuration](#building-the-global-configuration)

</div>

</nav>

<h2 id="basics">Basics</h2>

Configs allow you to store changeable values that power your application.  Unlike environment variables, they do not typically change between environments.  Configurations must implement `IConfiguration`, which provides the following methods:

```php
$config->getArray('foo');
         
$config->getBool('foo');

$config->getFloat('foo');

$config->getInt('foo');

$config->getString('foo');

$config->getValue('foo');

$config->tryGetArray('foo', $value);

$config->tryGetBool('foo', $value);

$config->tryGetFloat('foo', $value);

$config->tryGetInt('foo', $value);

$config->tryGetString('foo', $value);

$config->tryGetValue('foo', $value);
```

<h3 id="reading-php-files">Reading PHP Files</h3>

Let's say you have a PHP config array that looks like this:

```php
return [
    'api' => [
        'supportedLanguages' => ['en', 'es']
    ]
];
```

You can load this PHP file into a configuration object:

```php
use Aphiria\Configuration\PhpConfigurationFileReader;

$config = (new PhpConfigurationFileReader)->readConfiguration('config.php');
```

Grab the supported languages by using `.` as a delimiter for nested sections:

```php
$supportedLanguages = $config->getArray('api.supportedLanguages');
```

> **Note:**  Avoid using periods as keys in your configs.  If you must, you can change the delimiter character (eg to `:`) by passing it in as a second parameter to `readConfiguration()`.

<h3 id="reading-json-files">Reading JSON Files</h3>

Similarly, you can read JSON config files.

```php
use Aphiria\Configuration\JsonConfigurationFileReader;

$config = (new JsonConfigurationFileReader)->readConfiguration('config.json');
```

<h3 id="custom-file-readers">Custom File Readers</h3>

You can create your own custom file reader.  Let's look at an example that reads from YAML files:

```php
use Aphiria\Configuration\ConfigurationException;
use Aphiria\Configuration\HashTableConfiguration;
use Aphiria\Configuration\IConfiguration;
use Aphiria\Configuration\IConfigurationFileReader;
use Symfony\Component\Yaml\Yaml;

class YamlConfigurationFileReader implements IConfigurationFileReader
{
    public function readConfiguration(string $path, string $pathDelimiter = '.'): IConfiguration
    {
        if (!file_exists($path)) {
            throw new ConfigurationException("$path does not exist");
        }

        $hashTable = Yaml::parseFile($path);

        if (!\is_array($hashTable)) {
            throw new ConfigurationException("Configuration in $path must be an array");
        }

        return new HashTableConfiguration($hashTable, $pathDelimiter);
    }
}
```
  
<h2 id="global-configuration">Global Configuration</h2>

`GlobalConfiguration` is a static class that can access values from multiple sources registered via `GlobalConfiguration::addConfigurationSource()`.  It is the most convenient way to read configuration values from places like [binders](binders.md).  Let's look at its methods:

```php
use Aphiria\Configuration\GlobalConfiguration;

GlobalConfiguration::getArray('foo');

GlobalConfiguration::getBool('foo');

GlobalConfiguration::getFloat('foo');

GlobalConfiguration::getInt('foo');

GlobalConfiguration::getString('foo');

GlobalConfiguration::getValue('foo');

GlobalConfiguration::tryGetArray('foo', $value);

GlobalConfiguration::tryGetBool('foo', $value);

GlobalConfiguration::tryGetFloat('foo', $value);

GlobalConfiguration::tryGetInt('foo', $value);

GlobalConfiguration::tryGetString('foo', $value);

GlobalConfiguration::tryGetValue('foo', $value);
```

These methods mimic the `IConfiguration` interface, but are static.  Like `IConfiguration`, you can use `.` as a delimiter between sections.  If you have 2 configuration sources registered, `GlobalConfiguration` will attempt to find the path in the first registered source, and, if it's not found, the second source.  If no value is found, the `get*()` methods will throw a `ConfigurationException`, and the `tryGet*()` methods will return `false`.

<h3 id="building-the-global-configuration">Building The Global Configuration</h3>

`GlobalConfigurationBuilder` simplifies configuring different sources and building your global configuration.  In this example, we're loading from a PHP file, a JSON file, and from environment variables:

```php
use Aphiria\Configuration\GlobalConfigurationBuilder;

$globalConfigurationBuilder = new GlobalConfigurationBuilder();
$globalConfigurationBuilder->withPhpFileConfigurationSource('config.php')
    ->withJsonFileConfigurationSource('config.json')
    ->withEnvironmentVariables()
    ->build();
```

> **Note:** The reading of files and environment variables is deferred until `build()` is called.

After `build()` is called, you can start accessing `GlobalConfiguration`.

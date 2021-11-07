<h1 id="doc-title">Collections</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Introduction](#introduction)
2. [Key-Value Pairs](#key-value-pairs)
   1. [key](#key-value-pairs-key)
   2. [value](#key-value-pairs-value)
3. [Array Lists](#array-lists)
   1. [add()](#array-lists-add)
   2. [addRange()](#array-lists-add-range)
   3. [clear()](#array-lists-clear)
   4. [containsValue()](#array-lists-contains-value)
   5. [count()](#array-lists-count)
   6. [get()](#array-lists-get)
   7. [indexOf()](#array-lists-index-of)
   8. [insert()](#array-lists-insert)
   9. [intersect()](#array-lists-intersect)
   10. [removeIndex()](#array-lists-remove-index)
   11. [reverse()](#array-lists-reverse)
   12. [sort()](#array-lists-sort)
   12. [toArray()](#array-lists-to-array)
   12. [union()](#array-lists-union)
4. [Hash Tables](#hash-tables)
   1. [add()](#hash-tables-add)
   2. [addRange()](#hash-tables-add-range)
   3. [clear()](#hash-tables-clear)
   4. [containsKey()](#hash-tables-contains-key)
   5. [containsValue()](#hash-tables-contains-value)
   6. [count()](#hash-tables-count)
   7. [get()](#hash-tables-get)
   8. [getKeys()](#hash-tables-get-keys)
   9. [getValues()](#hash-tables-get-values)
   10. [removeKey()](#hash-tables-remove-key)
   11. [removeValue()](#hash-tables-remove-value)
   12. [toArray()](#hash-tables-to-array)
   13. [tryGet()](#hash-tables-try-get)
5. [Hash Sets](#hash-sets)
   1. [add()](#hash-sets-add)
   2. [addRange()](#hash-sets-add-range)
   3. [clear()](#hash-sets-clear)
   4. [containsValue()](#hash-sets-contains-value)
   5. [count()](#hash-sets-count)
   6. [intersect()](#hash-sets-intersect)
   7. [removeValue()](#hash-sets-remove-value)
   8. [sort()](#hash-sets-sort)
   9. [toArray()](#hash-sets-to-array)
   10. [union()](#hash-sets-union)
6. [Stacks](#stacks)
   1. [clear()](#stacks-clear)
   2. [containsValue()](#stacks-contains-value)
   3. [count()](#stacks-count)
   4. [peek()](#stacks-peek)
   5. [pop()](#stacks-pop)
   1. [push()](#stacks-push)
   1. [toArray()](#stacks-to-array)
7. [Queues](#queues)
   1. [clear()](#queues-clear)
   2. [containsValue()](#queues-contains-value)
   3. [count()](#queues-count)
   4. [dequeue()](#queues-dequeue)
   5. [enqueue()](#queues-enqueue)
   6. [peek()](#queues-peek)
   7. [toArray()](#queues-to-array)
8. [Immutable Array Lists](#immutable-array-lists)
   1. [containsValue()](#immutable-array-lists-contains-value)
   2. [count()](#immutable-array-lists-count)
   3. [get()](#immutable-array-lists-get)
   4. [indexOf()](#immutable-array-lists-index-of)
   5. [toArray()](#immutable-array-lists-to-array)
9. [Immutable Hash Tables](#immutable-hash-tables)
   1. [containsKey()](#immutable-hash-tables-contains-key)
   2. [containsValue()](#immutable-hash-tables-contains-value)
   3. [count()](#immutable-hash-tables-count)
   4. [get()](#immutable-hash-tables-get)
   5. [getKeys()](#immutable-hash-tables-get-keys)
   6. [getValues()](#immutable-hash-tables-get-values)
   7. [toArray()](#immutable-hash-tables-to-array)
   8. [tryGet()](#immutable-hash-tables-try-get)
10. [Immutable Hash Sets](#immutable-hash-sets)
    1. [containsValue()](#immutable-hash-sets-contains-value)
    2. [count()](#immutable-hash-sets-count)
    3. [toArray()](#immutable-hash-sets-to-array)

</div>

</nav>

<h2 id="introduction">Introduction</h2>

Unfortunately, PHP's support for collections is relatively incomplete.  The `array` type is reused for multiple types, like hash tables and lists.  PHP also has _some_ support for advanced types in the SPL library, but it is incomplete, and its syntax is somewhat clunky.  To cover for PHP's lack of coverage of collections, Aphiria provides simple wrappers for common collections found in most other programming languages.

<h2 id="key-value-pairs">Key-Value Pairs</h2>

Like its name implies, `KeyValuePair` holds a key and a value.  Unlike key-value pairs in native PHP arrays, keys in `KeyValuePair` can be any value, including an object.  To instantiate one, pass in the key and value:

```php
use Aphiria\Collections\KeyValuePair;

$kvp = new KeyValuePair('thekey', 'thevalue');
```

<h3 id="key-value-pairs-key">KeyValuePair::key</h3>

_Runtime: O(1)_

To get the key-value pair's key, call

```php
$kvp->key;
```

<h3 id="key-value-pairs-value">KeyValuePair::value</h3>

_Runtime: O(1)_

To get the key-value pair's value, call

```php
$kvp->value;
```

<h2 id="array-lists">Array Lists</h2>

Aphiria's `ArrayList` is probably the most similar to PHP's built-in indexed array functionality.  You can instantiate one with or without an array:

```php
use Aphiria\Collections\ArrayList;

$arrayList = new ArrayList();
// Or...
$arrayList = new ArrayList(['foo', 'bar']);
```

> **Note:** `ArrayList` implements `ArrayAccess` and `IteratorAggregate`, so you can use array-like accessors and iterate over it.

<h3 id="array-lists-add">add()</h3>

_Runtime: O(1)_

You can add a value via

```php
$arrayList->add('foo');
```

<h3 id="array-lists-add-range">addRange()</h3>

_Runtime: O(n), n = numbers of values added_

You can add multiple values at once:

```php
$arrayList->addRange(['foo', 'bar']);
```

<h3 id="array-lists-clear">clear()</h3>

_Runtime: O(1)_

You can remove all values in the array list:

```php
$arrayList->clear();
```

<h3 id="array-lists-contains-value">containsValue()</h3>

_Runtime: O(n)_

To check for a value, call

```php
$containsValue = $arrayList->containsValue('foo');
```

<h3 id="array-lists-count">count()</h3>

_Runtime: O(1)_

To grab the number of values in the array list, call

```php
$count = $arrayList->count();
```

<h3 id="array-lists-get">get()</h3>

_Runtime: O(1)_

To get the value at a certain index from an array list, call

```php
$value = $arrayList->get(123);
```

If the index is out of range, an `OutOfRangeException` will be thrown.

<h3 id="array-lists-index-of">indexOf()</h3>

_Runtime: O(n)_

To grab the index for a value, call

```php
$index = $arrayList->indexOf('foo');
```

<h3 id="array-lists-insert">insert()</h3>

_Runtime: O(1)_

To insert a value at a specific index, call

```php
$arrayList->insert(23, 'foo');
```

<h3 id="array-lists-intersect">intersect()</h3>

_Runtime: O(nm)_, n = number of values in the array list, and m = number of values in the parameter array

You can intersect an array list's values with an array by calling

```php
$intersectedList = $arrayList->intersect(['foo', 'bar']);
```

<h3 id="array-lists-remove-index">removeIndex()</h3>

_Runtime: O(1)_

To remove a value by index, call

```php
$arrayList->removeIndex(123);
```

To remove a specific value, call

```php
$arrayList->removeValue('foo');
```

<h3 id="array-lists-reverse">reverse()</h3>

_Runtime: O(n)_

To reverse the values in the list, call

```php
$reversedList = $arrayList->reverse();
```

<h3 id="array-lists-sort">sort()</h3>

_Runtime: O(n log n)_

You can sort values similar to the way you can sort PHP arrays via `usort()`:

```php
$sortedList = $arrayList->sort(fn ($a, $b) => $a <=> $b);
```

<h3 id="array-lists-to-array">toArray()</h3>

_Runtime: O(1)_

You can get the underlying array by calling

```php
$array = $arrayList->toArray();
```

<h3 id="array-lists-union">union()</h3>

_Runtime: O(nm)_, n = number of values in the array list, and m = number of values in the parameter array

You can union an array list's values with an array via

```php
$unionedList = $arrayList->union(['foo', 'bar']);
```

<h2 id="hash-tables">Hash Tables</h2>

Hash tables are most similar to PHP's built-in associative array functionality - they map keys to values.  Unlike PHP associative arrays (which only supports scalars as keys), Aphiria's `HashTables` support scalars, objects, arrays, and resources as keys.  You can instantiate one with or without an array of key-value pairs:

```php
use Aphiria\Collections\HashTable;

$hashTable = new HashTable();
// Or...
$hashTable = new HashTable([new KeyValuePair('foo', 'bar')]);
```

> **Note:** `HashTable` implements `ArrayAccess` and `IteratorAggregate`, so you can use array-like accessors and iterate over it. The keys will be numeric, and the values will be [key-value pairs](#key-value-pairs).

<h3 id="hash-tables-add">add()</h3>

_Runtime: O(1)_

To add a value, call

```php
$hashTable->add('foo', 'bar');
```

<h3 id="hash-tables-add-range">addRange()</h3>

_Runtime: O(n), n = number of values added_

To add multiple values at once, pass in an array of `KeyValuePair` objects:

```php
$kvps = [
    new KeyValuePair('foo', 'bar'),
    new KeyValuePair('baz', 'blah')
];
$hashTable->addRange($kvps);
```

<h3 id="hash-tables-clear">clear()</h3>

_Runtime: O(1)_

You can remove all values:

```php
$hashTable->clear();
```

<h3 id="hash-tables-contains-key">containsKey()</h3>

_Runtime: O(1)_

To check for a value, call

```php
$containsKey = $hashTable->containsKey('foo');
```

<h3 id="hash-tables-contains-value">containsValue()</h3>

_Runtime: O(n)_

To check for a key, call

```php
$containsValue = $hashTable->containsValue('foo');
```

<h3 id="hash-tables-count">count()</h3>

_Runtime: O(1)_

To get the number of values in the hash table, call

```php
$count = $hashTable->count();
```

<h3 id="hash-tables-get">get()</h3>

_Runtime: O(1)_

To get a value at a key, call

```php
$value = $hashTable->get('foo');
```

If the value does not exist, an `OutOfBoundsException` will be thrown.

<h3 id="hash-tables-get-keys">getKeys()</h3>

_Runtime: O(n)_

You can grab all of the keys in the hash table:

```php
$hashTable->getKeys();
```

<h3 id="hash-tables-get-values">getValues()</h3>

_Runtime: O(n)_

You can grab all of the values in the hash table:

```php
$hashTable->getValues();
```

<h3 id="hash-tables-remove-key">removeKey()</h3>

_Runtime: O(1)_

To remove a value at a certain key, call

```php
$hashTable->removeKey('foo');
```

<h3 id="hash-tables-remove-value">removeValue()</h3>

_Runtime: O(n)_

To remove a value, call

```php
$hashTable->removeValue('foo');
```

<h3 id="hash-tables-to-array">toArray()</h3>

_Runtime: O(n)_

To get the underlying array, call

```php
$array = $hashTable->toArray();
```

This will return a list of `KeyValuePair` - not an associative array.  The reason for this is that keys can be non-strings, which is not supported in PHP.

<h3 id="hash-tables-try-get">tryGet()</h3>

_Runtime: O(1)_

If you would like to try to safely get a value that may or may not exist, use `tryGet()`.  It'll return `true` if the key exists, otherwise `false`.  It will also set the second parameter to the value if the key exists.

```php
$value = null;
$exists = $hashTable->tryGet('foo', $value);
```

<h2 id="hash-sets">Hash Sets</h2>

Hash sets are lists with unique values.  They accept objects, scalars, arrays, and resources as values.  You can instantiate one with or without an array of key => value pairs:

```php
use Aphiria\Collections\HashSet;

$set = new HashSet();
// Or...
$set = new HashSet(['foo', 'bar']);
```

> **Note:** `HashSet` implements `IteratorAggregate`, so you can iterate over it.

<h3 id="hash-sets-add">add()</h3>

_Runtime: O(1)_

You can add a value via

```php
$set->add('foo');
```

<h3 id="hash-sets-add-range">addRange()</h3>

_Runtime: O(n), n = number of values added_

You can add multiple values at once:

```php
$set->addRange(['foo', 'bar']);
```

<h3 id="hash-sets-clear">clear()</h3>

_Runtime: O(1)_

To remove all values in the set, call `clear()`:

```php
$set->clear();
```

<h3 id="hash-sets-contains-value">containsValue()</h3>

_Runtime: O(1)_

To check for a value, call

```php
$containsValue = $set->containsValue('foo');
```

<h3 id="hash-sets-count">count()</h3>

_Runtime: O(1)_

To grab the number of values in the hash set, call

```php
$count = $set->count();
```

<h3 id="hash-sets-intersect">intersect()</h3>

_Runtime: O(nm)_

You can intersect a hash set with an array by calling

```php
$intersectedSet = $set->intersect(['foo', 'bar']);
```

<h3 id="hash-sets-remove-value">removeValue()</h3>

_Runtime: O(1)_

To remove a specific value, call

```php
$set->removeValue('foo');
```

<h3 id="hash-sets-sort">sort()</h3>

_Runtime: O(n log n)_

You can sort values similar to the way you can sort PHP arrays via `usort()`:

```php
$sortedSet = $set->sort(fn ($a, $b) => $a <=> $b);
```

<h3 id="hash-sets-to-array">toArray()</h3>

_Runtime: O(n)_

To get the underlying array, call

```php
$array = $set->toArray();
```

<h3 id="hash-sets-union">union()</h3>

_Runtime: O(nm)_

You can union a hash set with an array via

```php
$unionedSet = $set->union(['foo', 'bar']);
```

<h2 id="stacks">Stacks</h2>

Stacks are first-in, last-out (FILO) data structures.  To create one, call

```php
use Aphiria\Collections\Stack;

$stack = new Stack();
```

> **Note:** `Stack` implements `IteratorAggregate`, so you can iterate over it.

<h3 id="stacks-clear">clear()</h3>

_Runtime: O(1)_

To clear the values in the stack, call

```php
$stack->clear();
```

<h3 id="stacks-contains-value">containsValue()</h3>

_Runtime: O(n)_

To check for a value within a stack, call

```php
$containsValue = $stack->containsValue('foo');
```

<h3 id="stacks-count">count()</h3>

_Runtime: O(1)_

To get the number of values in the stack, call

```php
$count = $stack->count();
```

<h3 id="stacks-peek">peek()</h3>

_Runtime: O(1)_

To peek at the top value in the stack, call

```php
$value = $stack->peek();
```

<h3 id="stacks-pop">pop()</h3>

_Runtime: O(1)_

To pop a value off the stack, call

```php
$value = $stack->pop();
```

If there are no values in the stack, this will return `null`.

<h3 id="stacks-push">push()</h3>

_Runtime: O(1)_

To push a value onto the stack, call

```php
$stack->push('foo');
```

If there are no values in the stack, this will return `null`.

<h3 id="stacks-to-array">toArray()</h3>

_Runtime: O(1)_

To get the underlying array, call

```php
$array = $stack->toArray();
```

<h2 id="queues">Queues</h2>

Queues are first-in, first-out (FIFO) data structures.  To create one, call

```php
use Aphiria\Collections\Queue;

$queue = new Queue();
```

> **Note:** `Queue` implements `IteratorAggregate`, so you can iterate over it.

<h3 id="queues-clear">clear()</h3>

_Runtime: O(1)_

To clear the queue, call

```php
$queue->clear();
```

<h3 id="queues-contains-value">containsValue()</h3>

_Runtime: O(n)_

To check for a value within a queue, call

```php
$containsValue = $queue->containsValue('foo');
```

<h3 id="queues-count">count()</h3>

_Runtime: O(1)_

To get the number of values in the queue, call

```php
$count = $queue->count();
```

<h3 id="queues-dequeue">dequeue()</h3>

_Runtime: O(1)_

To dequeue a value from the queue, call

```php
$value = $queue->dequeue();
```

If there are no values in the queue, this will return `null`.

<h3 id="queues-enqueue">enqueue()</h3>

_Runtime: O(1)_

To enqueue a value onto the queue, call

```php
$queue->enqueue('foo');
```

<h3 id="queues-peek">peek()</h3>

_Runtime: O(1)_

To peek at the value at the beginning of the queue, call

```php
$value = $queue->peek();
```

If there are no values in the queue, this will return `null`.

<h3 id="queues-to-array">toArray()</h3>

_Runtime: O(1)_

To get the underlying array, call

```php
$array = $queue->toArray();
```

<h2 id="immutable-array-lists">Immutable Array Lists</h2>

`ImmutableArrayList` are read-only [array lists](#array-lists).  To instantiate one, pass in the array of values:

```php
use Aphiria\Collections\ImmutableArrayList;

$arrayList = new ImmutableArrayList(['foo', 'bar']);
```

> **Note:** `ImmutableArrayList` implements `ArrayAccess` and `IteratorAggregate`, so you can use array-like accessors and iterate over it.

<h3 id="immutable-array-lists-contains-value">ImmutablecontainsValue()</h3>

_Runtime: O(n)_

To check for a value, call

```php
$containsValue = $arrayList->containsValue('foo');
```

<h3 id="immutable-array-lists-count">Immutablecount()</h3>

_Runtime: O(1)_

To grab the number of values in the array list, call

```php
$count = $arrayList->count();
```

<h3 id="immutable-array-lists-get">Immutableget()</h3>

_Runtime: O(1)_

To get the value at a certain index from an array list, call

```php
$value = $arrayList->get(123);
```

If the index is out of range, an `OutOfRangeException` will be thrown.

<h3 id="immutable-array-lists-index-of">ImmutableindexOf()</h3>

_Runtime: O(n)_

To grab the index for a value, call

```php
$index = $arrayList->indexOf('foo');
```

If the array list doesn't contain the value, `null` will be returned.

<h3 id="immutable-array-lists-to-array">ImmutabletoArray()</h3>

_Runtime: O(1)_

If you want to grab the underlying array, call

```php
$array = $arrayList->toArray();
```

<h2 id="immutable-hash-tables">Immutable Hash Tables</h2>

Sometimes, your business logic might dictate that a [hash table](#hash-tables) is read-only.  Aphiria provides support via `ImmutableHashTable`.  It requires that you pass key-value pairs into its constructor:

```php
use Aphiria\Collections\ImmutableHashTable;

$hashTable = new ImmutableHashTable([new KeyValuePair('foo', 'bar')]);
```

> **Note:** `ImmutableHashTable` implements `ArrayAccess` and `IteratorAggregate`, so you can use array-like accessors and iterate over it. When iterating, the keys will be numeric, and the values will be [key-value pairs](#key-value-pairs).

<h3 id="immutable-hash-tables-contains-key">ImmutablecontainsKey()</h3>

_Runtime: O(1)_

To check for a key, call

```php
$containsKey = $hashTable->containsKey('foo');
```

<h3 id="immutable-hash-tables-contains-value">ImmutablecontainsValue()</h3>

_Runtime: O(n)_

To check for a value, call

```php
$containsValue = $hashTable->containsValue('foo');
```

<h3 id="immutable-hash-tables-count">Immutablecount()</h3>

_Runtime: O(1)_

To get the number of values in the hash table, call

```php
$count = $hashTable->count();
```

<h3 id="immutable-hash-tables-get">Immutableget()</h3>

_Runtime: O(1)_

To get a value at a key, call

```php
$value = $hashTable->get('foo');
```

If the value does not exist, an `OutOfBoundsException` will be thrown.

<h3 id="immutable-hash-tables-get-keys">ImmutablegetKeys()</h3>

_Runtime: O(n)_

You can grab all of the keys in the hash table:

```php
$hashTable->getKeys();
```

<h3 id="immutable-hash-tables-get-values">ImmutablegetValues()</h3>

_Runtime: O(n)_

You can grab all of the values in the hash table:

```php
$hashTable->getValues();
```

<h3 id="immutable-hash-tables-to-array">ImmutabletoArray()</h3>

_Runtime: O(n)_

To get the underlying array, call

```php
$array = $hashTable->toArray();
```

This will return a list of `KeyValuePair` - not an associative array.  The reason for this is that keys can be non-strings (eg objects) in hash tables, but keys in PHP associative arrays must be serializable.

<h3 id="immutable-hash-tables-try-get">ImmutabletryGet()</h3>

_Runtime: O(1)_

If you would like to try to safely get a value that may or may not exist, use `tryGet()`.  It'll return `true` if the key exists, otherwise `false`.  It will also set the second parameter to the value if the key exists.

```php
$value = null;
$exists = $hashTable->tryGet('foo', $value);
```

<h2 id="immutable-hash-sets">Immutable Hash Sets</h2>

Immutable hash sets are read-only [hash sets](#hash-sets).  They accept objects, scalars, arrays, and resources as values.  You can instantiate one with a list of values:

```php
use Aphiria\Collections\ImmutableHashSet;

$set = new ImmutableHashSet(['foo', 'bar']);
```

> **Note:** `ImmutableHashSet` implements `IteratorAggregate`, so you can iterate over it.

<h3 id="immutable-hash-sets-contains-value">ImmutablecontainsValue()</h3>

_Runtime: O(1)_

To check for a value, call

```php
$containsValue = $set->containsValue('foo');
```

<h3 id="immutable-hash-sets-count">Immutablecount()</h3>

_Runtime: O(1)_

To grab the number of values in the set, call

```php
$count = $set->count();
```

<h3 id="immutable-hash-sets-to-array">ImmutabletoArray()</h3>

_Runtime: O(n)_

To get the underlying array, call

```php
$array = $set->toArray();
```

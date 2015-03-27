Json Match
====================

[![Build Status](https://travis-ci.org/fesor/json_matcher.svg?branch=master)](https://travis-ci.org/fesor/json_matcher) [![Latest Stable Version](https://poser.pugx.org/fesor/json_matcher/v/stable.svg)](https://packagist.org/packages/fesor/json_matcher) [![Latest Unstable Version](https://poser.pugx.org/fesor/json_matcher/v/unstable.svg)](https://packagist.org/packages/fesor/json_matcher) [![License](https://poser.pugx.org/fesor/json_matcher/license.svg)](https://packagist.org/packages/fesor/json_matcher) [![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/fesor/json_matcher/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/fesor/json_matcher/?branch=master) [![Total Downloads](https://poser.pugx.org/fesor/json_matcher/downloads.svg)](https://packagist.org/packages/fesor/json_matcher)

If you tried to test your JSON based REST APIs, then you probably faced a several issues:

- You can't simply check is a response is equal to given string as there are things like server-generated IDs and timestamps.
- Key ordering should be the same both for your API and for expected JSON.
- Matching the whole responses breaks DRY for the spec

All this issues can be solved with two simple things: JSON normalization and key exclusion on matching.

## Getting started

You can install this library via composer:
```
composer require fesor/json_matcher
```

Then you will need an `JsonMatcher` instance to be created. To do that, you have three possible options:
 - manually create instance with all dependencies and set subject
 - use JsonMatcherFactory. This is useful when you have some IoC container. In this case you'll need to register this class as a service.
 - use named constructor `JsonMatcher::create` as shortcut. This method will handle all dependencies for you.

Matchers should have a subject, on which matching will be performed. You can set subject via `setSubject` method. If you are using matcher factory or named constructor, then you don't have to call setSubject but you can change it with it.

Example:
```php
$jsonResponseSubject = JsonMatcher::create($myJsonResponse);

// or you can use factory instead
$jsonResponseSubject = $matcherFactory->create($myJsonResponse);

// and there you go, for example you may use something like this 
// for your gherkin steps implementations
$jsonResponseSubject
    ->haveSize(1, ['at' => 'friends']) // check that list of friends was incremented
    ->includes($friendJson, ['at' => 'friends']) // check that correct record contained in collection
;
```

You can provide list of by-default excluded keys as second argument in constructors:
```php
$matcher = JsonMatcher::create($subject, ['id', 'created_at']);
```

Please not, that `id` key will be ignored by default.

## Matchers

All matchers are chainable, supports negative matching and some options. See detailed description for more information.

### equal
This is most common matcher of all. You take two json strings and compares it. Except that before compassion this matcher will normalize structure of both JSON strings, will reorder keys, exclude some of them (this is configurable) and then will simply assert that both strings are equal. You can specify list of excluded keys with `excluding` options:
```php
$jsonResponse = '["id": 1, "json": "spec"]';
$expectedJson = '["json": "spec"]';
$matcher
    ->setSubject($jsonResponse)
    ->equal($expectedJson, ['excluding' => ['id']])
;
```

If you have some keys, which contains some time dependent value of some server-generated IDs it is more convenient to specify list of excluded-by-default keys when you construct matcher object:
```php
$matcher = JsonMatcher::create($subject, ['id', 'created_at', 'updated_at']);
```

If you want the values for these keys were taken into account during the matching, you can specify list of included keys with `including` options
```php
$matcher = JsonMatcher::create($jsonResponse, ['id', 'created_at', 'updated_at']);
$jsonResponseSubject->equal($expectedJson, ['including' => ['id']]);
```

Also you can specify json path on which matching should be done via `at` options. We will back to this later since all matchers supports this option.

### includes
This matcher is pretty match the same as `equal` matcher except that is recursively scan given JSON and tries to find expected JSON in any values. This is useful for cases when you checking that some record exists in collection and you do not know or don't whant to know specific path to it.

```php
$json = <<<JSON
{
    "collection": [
        "json",
        "matcher"
    ]
}
JSON;

$needle = '"matcher"';
$matcher
    ->setSubject($json)
    ->includes($needle, ['at' => 'collection'])
;
```

This matcher works the same way as `equal` matcher, so it accepts the same options.

### havePath
This matcher checks if given JSON have specific path ot not.

```php
$json = <<<JSON
{
    "collection": [
        "json",
        "matcher"
    ]
}
JSON;

$matcher
    ->setSubject($json)
    ->havePath('collection/1')
;
```

### haveSize
This matcher checks is collection in given JSON contains specific amount of entities.

```php
$json = <<<JSON
{
    "collection": [
        "json",
        "matcher"
    ]
}
JSON;

$matcher
    ->setSubject($json)
    ->haveSize(2, ['at' => 'collection'])
;
```

### haveType
```php
$json = <<<JSON
{
    "collection": [
        {},
        "json",
        42,
        13.45
    ]
}
JSON;

$matcher
    ->setSubject($json)
    ->haveType('array', ['at' => 'collection'])
    ->haveType('object', ['at' => 'collection/0'])
    ->haveType('string', ['at' => 'collection/1'])
    ->haveType('integer', ['at' => 'collection/2'])
    ->haveType('float', ['at' => 'collection/3'])
;
```

### Negative matching
To invert expectations just call matcher methods with `not` prefix:
```php
$matcher
    ->setSubject($json)
    ->notEqual($expected)
    ->notIncludes($part)
;
```

### Json Path
Also all methods have option, which specifies path which should be performed matching. For example:

```php
$json = <<<JSON
{
    "collection": [
        "item"
    ]
}
JSON;
$expected = '"item"';
JsonMatcher::create($actual)
    ->equal($expected, ['at' => 'collection/0'])
;
```

# Extending the Hax Compiler

Yuga's Hax compiler supports custom directives and compiler extensions.

## Custom directives

Register a directive on the compiler:

```php
view()->compiler()->directive('datetime', function ($expression) {
    return "<?php echo date('Y-m-d H:i', {$expression}); ?>";
});
```

It can then be used in Hax:

```php
@datetime($createdAt)
```

## Compiler extensions

Packages that introduce syntax beyond `@directives` can register a compiler pass:

```php
view()->compiler()->extend(
    function (string $value, $compiler): string {
        return $value;
    }
);
```

Extensions receive the template source and active compiler and return transformed source.

This keeps package syntax outside Yuga's core compiler. Yuga Live Components uses this mechanism for `<ylc:mount ... />`.

## Attribute parsing

The compiler exposes attribute parsing and compilation helpers, allowing extensions to share Hax component conventions:

```html
title="Report"
:year="$year"
disabled
```

A leading `:` marks a PHP expression; unbound values are literals.

## Rendering architecture

```text
Factory
  -> Engine
      -> Finder
      -> Compiler (.hax.php)
      -> evaluated PHP
```

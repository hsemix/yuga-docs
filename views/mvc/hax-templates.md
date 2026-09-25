---
description: Hax is Yuga's simple, compiled PHP template engine
---

# Hax Templates

Hax is Yuga's template engine for MVC views. It adds convenient template syntax while still allowing ordinary PHP when you need it.

Hax templates use the `.hax.php` extension and are normally stored in `resources/views`. They are compiled to plain PHP and cached, so the template syntax does not need to be interpreted on every render.

```text
resources/
└── views/
    ├── layouts/
    │   └── app.hax.php
    ├── components/
    │   └── alert.hax.php
    └── users/
        └── profile.hax.php
```

A view can be rendered with:

```php
return view('users.profile', [
    'user' => $user,
]);
```

## Displaying data

Use double curly braces for escaped output:

```html
<h1>Hello, {{ $user->name }}</h1>
```

Hax escapes this output with `htmlspecialchars`.

For trusted content that should not be escaped, use raw echo syntax:

```html
{!! $html !!}
```

Hax also supports a default-value form for variables:

```html
{{ $name or 'Guest' }}
```

## Comments

Hax comments are removed during compilation:

```html
{{-- This will not be included in the rendered HTML. --}}
```

## Template inheritance

Layouts allow several pages to share the same structure.

### Defining a layout

```html
<!-- resources/views/layouts/app.hax.php -->

<!doctype html>
<html>
<head>
    <title>@yield('title')</title>
</head>
<body>
    <main>
        @yield('content')
    </main>
</body>
</html>
```

### Extending a layout

```php
@extends('layouts.app')

@section('title')
    User Profile
@endsection

@section('content')
    <h1>{{ $user->name }}</h1>
@endsection
```

### Parent section content

Use `@parent` when a child section should retain content defined by its parent:

```php
@section('sidebar')
    @parent

    <a href="/profile">Profile</a>
@endsection
```

## Includes

Use `@include` to render another view:

```php
@include('partials.header')
```

The included view receives the variables currently defined in the parent template.

You can also pass additional data:

```php
@include('partials.user', ['user' => $user])
```

## Conditionals

Hax conditionals map directly to PHP conditionals:

```php
@if ($user->isAdmin())
    <p>Administrator</p>
@elseif ($user->isModerator())
    <p>Moderator</p>
@else
    <p>User</p>
@endif
```

## Loops

### Foreach

```php
@foreach ($users as $user)
    <p>{{ $user->name }}</p>
@endforeach
```

### For

```php
@for ($i = 0; $i < 10; $i++)
    <span>{{ $i }}</span>
@endfor
```

### While

```php
@while ($condition)
    ...
@endwhile
```

## Raw PHP

For small pieces of PHP, pass an expression directly:

```php
@php($total = count($items))
```

For a PHP block:

```php
@php
    $total = count($items);
    $label = 'items';
@endphp
```

Because Hax compiles to PHP, ordinary PHP can also be used in a template when appropriate.

## Components

Reusable view components use the `x-` syntax.

### Self-closing components

```html
<x-alert type="success" :message="$message" />
```

Literal attributes are strings. Prefix an attribute with `:` to treat its value as a PHP expression.

Boolean attributes are also supported:

```html
<x-button disabled />
```

### Components with content

```html
<x-card title="Account">
    <p>Account details</p>
</x-card>
```

The wrapped content becomes the component's default slot.

### Named slots

```html
<x-card>
    <x-slot:header>
        Account
    </x-slot:header>

    <p>Account details</p>
</x-card>
```

For more detail, see [Layouts, Components, Slots, and Fragments](layouts-components.md).

## Fragments

A Hax template can define named fragments:

```html
<fragment name="results">
    @foreach ($results as $result)
        <div>{{ $result->name }}</div>
    @endforeach
</fragment>
```

Fragments allow the view engine and packages built on it to address a specific rendered portion of a view.

## Stacks

Stacks are useful for content that a child view or component wants to contribute to another location, such as scripts or styles.

Push content:

```php
@push('scripts')
    <script src="/js/dashboard.js"></script>
@endpush
```

Render the stack:

```php
@stack('scripts')
```

Content can be inserted before existing pushed content with:

```php
@prepend('scripts')
    <script src="/js/bootstrap.js"></script>
@endprepend
```

## Render once

Use `@once` when markup should only be emitted once during a render:

```php
@once('chart-library')
    <script src="/js/chart.js"></script>
@endonce
```

An identifier can be omitted:

```php
@once
    ...
@endonce
```

## Slots

Hax also provides section-manager slots:

```php
@slot('toolbar')
    <button>Save</button>
@endslot
```

Render a slot with:

```php
@yieldSlot('toolbar')
```

These are separate from the `<x-slot:name>` syntax used by view components.

## JSON

Use `@json` to JSON-encode a PHP value:

```html
<script>
    const user = @json($user);
</script>
```

## Forms

### CSRF field

```html
<form method="POST">
    @csrf
</form>
```

### HTTP method field

```html
<form method="POST">
    @method('PUT')
</form>
```

### Conditional attributes

```html
<input type="checkbox" @checked($enabled)>

<option @selected($selected)>Uganda</option>

<button @disabled($processing)>Submit</button>
```

## Conditional classes

`@class` builds a class attribute from an array:

```php
@class([
    'btn',
    'btn-primary' => $primary,
    'disabled' => $disabled,
])
```

## Conditional styles

`@style` provides the equivalent helper for inline styles:

```php
@style([
    'display: none' => $hidden,
    'font-weight: bold' => $important,
])
```

## Custom directives

Applications and packages can register their own `@directives`:

```php
view()->compiler()->directive('datetime', function ($expression) {
    return "<?php echo date('Y-m-d H:i', {$expression}); ?>";
});
```

Then use the directive normally:

```php
@datetime($createdAt)
```

## Extending Hax

Packages can add syntax that is not naturally represented by an `@directive` through compiler extensions:

```php
view()->compiler()->extend(
    function (string $value, $compiler): string {
        return $value;
    }
);
```

For example, Yuga Live Components registers its `<ylc:mount ... />` syntax this way instead of adding YLC-specific behavior to the Yuga framework.

See [Extending the Hax Compiler](extending-hax.md) for package integration details.

## Compilation order

The current compiler processes Hax source through these major passes:

1. comments
2. fragments
3. components
4. echos
5. statements/directives
6. registered compiler extensions

The resulting PHP is cached and evaluated by the view engine.

## Related documentation

- [Rendering Views](rendering.md)
- [Layouts, Components, Slots, and Fragments](layouts-components.md)
- [View Namespaces and Composers](namespaces-composers.md)
- [Extending the Hax Compiler](extending-hax.md)

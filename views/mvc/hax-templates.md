---
description: Hax is Yuga's compiled, HTML-first PHP template engine
---

# Hax Templates

Hax is Yuga's template engine for MVC views. It provides convenient template syntax while remaining close to HTML and PHP.

Hax templates use the `.hax.php` extension and are normally stored in `resources/views`. They are compiled to plain PHP and cached.

```text
resources/
└── views/
    ├── layouts/
    │   └── app.hax.php
    ├── components/
    │   ├── alert.hax.php
    │   └── form/
    │       └── input.hax.php
    └── users/
        └── profile.hax.php
```

```php
return view('users.profile', ['user' => $user]);
```

## Displaying data

Escaped output:

```html
<h1>Hello, {{ $user->name }}</h1>
{{ $name or 'Guest' }}
```

Raw output for trusted content:

```html
{!! $html !!}
```

Comments are removed during compilation:

```html
{{-- This will not be included in the rendered HTML. --}}
```

## Template inheritance

```html
@extends('layouts.app')

@section('title')
    User Profile
@endsection

@section('content')
    <h1>{{ $user->name }}</h1>
@endsection
```

A layout renders sections with:

```html
<title>@yield('title')</title>
<main>@yield('content')</main>
```

Use `@parent` when a child section should retain parent content.

## Includes

```php
@include('partials.header')
@include('partials.user', ['user' => $user])
```

Included views receive the variables currently defined in the calling template.

## Conditionals and loops

```php
@if ($user->isAdmin())
    <p>Administrator</p>
@elseif ($user->isModerator())
    <p>Moderator</p>
@else
    <p>User</p>
@endif

@foreach ($users as $user)
    <p>{{ $user->name }}</p>
@endforeach

@for ($i = 0; $i < 10; $i++)
    <span>{{ $i }}</span>
@endfor

@while ($condition)
    ...
@endwhile
```

## Raw PHP

```php
@php($total = count($items))

@php
    $total = count($items);
    $label = 'items';
@endphp
```

## Components

Application components are resolved beneath `resources/views/components`.

```html
<x-alert type="success" :message="$message" />
<x-button disabled />
```

Literal attributes are strings, `:` marks a PHP expression, and boolean attributes are supported.

Components may wrap content:

```html
<x-card title="Account">
    <p>Account details</p>
</x-card>
```

The wrapped content becomes `$slot`.

Nested components use dot notation:

```html
<x-form.input />
```

which resolves to `components/form/input.hax.php`.

Named slots are supported:

```html
<x-card>
    <x-slot:header>
        Account
    </x-slot:header>

    <p>Account details</p>
</x-card>
```

Namespaced package components use the same namespace registry as views:

```html
<x-blog::card />
<x-blog::forms.input />
```

See [Layouts, Components, Slots, and Fragments](layouts-components.md) for component attributes and the attribute bag.

## Fragments

```html
<fragment name="results">
    @foreach ($results as $result)
        <div>{{ $result->name }}</div>
    @endforeach
</fragment>
```

A fragment can be returned independently with `view(...)->fragment('results')`. Fragments are a core Views feature and do not require Yuga Live Components.

## Stacks and render-once blocks

```php
@push('scripts')
    <script src="/js/dashboard.js"></script>
@endpush

@prepend('scripts')
    <script src="/js/bootstrap.js"></script>
@endprepend

@stack('scripts')

@once('chart-library')
    <script src="/js/chart.js"></script>
@endonce
```

The `@once` identifier may be omitted.

## View slots

Section-manager slots are separate from component named slots:

```php
@slot('toolbar')
    <button>Save</button>
@endslot

@yieldSlot('toolbar')
```

## Helper directives

```html
@json($user)
@csrf
@method('PUT')

<input type="checkbox" @checked($enabled)>
<option @selected($selected)>Uganda</option>
<button @disabled($processing)>Submit</button>
```

Conditional classes and styles:

```php
@class([
    'btn',
    'btn-primary' => $primary,
    'disabled' => $disabled,
])

@style([
    'display: none' => $hidden,
    'font-weight: bold' => $important,
])
```

## Custom directives

```php
view()->compiler()->directive('datetime', function ($expression) {
    return "<?php echo date('Y-m-d H:i', {$expression}); ?>";
});
```

```php
@datetime($createdAt)
```

## Extending Hax

Packages can register transformations that operate on the full template:

```php
view()->compiler()->extend(
    function (string $value, $compiler): string {
        return $value;
    }
);
```

This keeps package-specific syntax outside Yuga's core compiler. See [Extending the Hax Compiler](extending-hax.md).

## Compilation order

The compiler currently processes:

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
- [View Namespaces, Shared Data, and Composers](namespaces-composers.md)
- [Extending the Hax Compiler](extending-hax.md)

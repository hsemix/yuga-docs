# Layouts, Components, Slots, and Fragments

## Layout inheritance

A child view can select a layout with `@extends` and define sections with `@section`.

```php
@extends('layouts.app')

@section('title')
Dashboard
@endsection

@section('content')
    <h1>Dashboard</h1>
@endsection
```

A layout renders a section with:

```php
@yield('content')
```

Use `@parent` inside a child section to retain content already defined by the parent.

## View components

Reusable anonymous components live in the `components` directory and use the `x-` syntax:

```text
resources/views/components/alert.hax.php
```

```html
<x-alert type="success" :message="$message" />
```

Literal attributes are strings. Prefix an attribute with `:` to pass a PHP expression. Boolean attributes are also supported.

Components can wrap content:

```html
<x-card title="Account">
    <p>Card body</p>
</x-card>
```

The wrapped content is available to the component as `$slot`.

### Nested components

Dot notation maps component names to nested directories:

```html
<x-form.input />
```

This resolves to:

```text
resources/views/components/form/input.hax.php
```

### Component attributes

Attributes are available as individual component variables and through the `$attributes` attribute bag:

```html
<button {{ $attributes }}>
    {!! $slot !!}
</button>
```

The attribute bag supports:

- `all()`
- `get()`
- `has()`
- `only()`
- `except()`
- `merge()`
- `class()`

For example:

```html
<button {{ $attributes->merge(['type' => 'button'])->class([
    'btn',
    'btn-disabled' => $disabled ?? false,
]) }}>
    {!! $slot !!}
</button>
```

Attributes supplied by the component caller override defaults passed to `merge()`.

### Component scope

Components are rendered as separate views. They receive data through attributes and slots rather than automatically inheriting every variable from the calling view.

## Named component slots

```html
<x-card>
    <x-slot:header>
        Account
    </x-slot:header>

    <p>Card body</p>
</x-card>
```

The component receives `$header` as well as the default `$slot`.

## Namespaced components

Components use the same namespace registry as ordinary views.

```html
<x-blog::card />
<x-blog::forms.input />
```

These resolve to:

```text
blog::components.card
blog::components.forms.input
```

This means packages do not need a separate component namespace registry. Once a view namespace is registered, its `components` directory is automatically addressable through `<x-namespace::...>`.

## View slots

Hax also provides section-manager slots, which are separate from component slots:

```php
@slot('toolbar')
    <button>Save</button>
@endslot

@yieldSlot('toolbar')
```

## Stacks

Push content into a named stack:

```php
@push('scripts')
    <script src="/js/dashboard.js"></script>
@endpush
```

Prepend content with:

```php
@prepend('scripts')
    <script src="/js/runtime.js"></script>
@endprepend
```

Render the stack with:

```php
@stack('scripts')
```

Use `@once` when markup should only be emitted once during a render:

```php
@once('chart-library')
    <script src="/js/chart.js"></script>
@endonce
```

## Fragments

Hax supports named fragments:

```html
<fragment name="results">
    <div>...</div>
</fragment>
```

A view can return only a named fragment:

```php
return view('search.results', [
    'results' => $results,
])->fragment('results');
```

Fragments belong to the Yuga view engine itself. They can be used without Yuga Live Components, while packages such as YLC can build partial-update behavior on top of them.

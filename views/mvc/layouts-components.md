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

### Component attributes and props

Component attributes are parsed as Hax attributes before the component is rendered. Literal values are passed as strings:

```html
<x-card title="Account" class="dashboard-card" />
```

Prefix an attribute with `:` to bind its value as a PHP expression:

```html
<x-card :user="$user" />
```

Bound attributes support normal PHP expressions, including property and method chains, comparisons, arrays, and arrow functions:

```html
<x-card
    :user="$account->owner"
    :visible="$score > 10"
    :options="['compact' => true, 'limit' => 10]"
    :filter="fn ($item) => $item->active"
/>
```

Attribute values remain quoted at the Hax level. The quotes delimit the attribute; for a bound attribute, the contents are emitted as PHP rather than passed as a string.

Boolean attributes can be written without a value:

```html
<x-button disabled required />
```

Literal quoted values may contain characters that are meaningful to PHP or HTML-like syntax, including `>`:

```html
<x-card title="A > B" />
```

Escaped quotes are supported inside quoted values:

```html
<x-card title="He said \"hello\"" />
```

Multiline component declarations are supported and are useful for larger prop sets:

```html
<x-card
    class="dashboard-card"
    :user="$account->owner"
    :visible="$score > 10"
>
    <p>Card body</p>
</x-card>
```

Bound prop expressions are evaluated in the calling view before the component starts rendering. An exception raised while evaluating a prop therefore belongs to the calling Hax view. Once component rendering begins, errors inside the component are reported against the component's own Hax source.

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

### Nested components

Dot notation maps component names to nested directories:

```html
<x-form.input />
```

This resolves to:

```text
resources/views/components/form/input.hax.php
```

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

## Component error context

Hax preserves source line positions during compilation so runtime errors can be reported against the original `.hax.php` source instead of only the generated PHP.

For example, if a bound prop fails:

```html
<x-card
    class="dashboard-card"
    :user="$account->owner->thisMethodDoesNotExist()"
>
```

the error is associated with the prop's line in the calling Hax view.

For errors raised while rendering a component, Yuga also keeps component invocation context. In development, the Hax debugging information can show the failing Hax source together with the component trace, making nested component failures easier to follow.

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

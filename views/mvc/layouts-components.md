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

Use `@parent` inside a child section to include the parent section content.

## View components

Reusable view components use the `x-` syntax:

```html
<x-alert type="success" :message="$message" />
```

Literal attributes are passed as strings. Prefix an attribute with `:` to pass a PHP expression.

Components can also wrap content:

```html
<x-card title="Account">
    <p>Card body</p>
</x-card>
```

The wrapped content is available as `$slot`.

## Named slots

```html
<x-card>
    <x-slot:header>
        Account
    </x-slot:header>

    <p>Card body</p>
</x-card>
```

## Fragments

Hax supports named fragments:

```html
<fragment name="results">
    <div>...</div>
</fragment>
```

Fragments can be rendered independently and are also useful to packages such as Yuga Live Components for partial updates.

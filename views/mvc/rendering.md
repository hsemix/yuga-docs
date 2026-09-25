# Rendering Views

Yuga's view layer is built around the view factory, engine, finder, Hax compiler, and supporting managers.

Views are normally stored in `resources/views`. Both `.hax.php` and plain `.php` files are supported.

## Rendering a view

```php
return view('users.profile', ['user' => $user]);
```

Dot, slash, and backslash notation are normalized by the view finder.

## Passing data

Data passed to a view is available directly in the template:

```php
return view('welcome', ['name' => 'Hamid']);
```

```html
<h1>Hello {{ $name }}</h1>
```

Data can also be attached fluently:

```php
return view('welcome')
    ->with('name', 'Hamid')
    ->with('title', 'Welcome');
```

Dynamic `withXxx` methods are supported:

```php
return view('profile')->withUser($user);
```

## Includes and scope

```php
@include('partials.header')
```

Includes are rendered as partial views and receive the variables currently defined in the calling template.

This differs from components, which are isolated views and receive their data through attributes and slots.

## Fragment rendering

A view containing a named fragment can return only that fragment:

```php
return view('users.index', [
    'users' => $users,
])->fragment('users-table');
```

A View may also be rendered explicitly with `render()` or `asString()`, and is stringable.

## View lookup

The finder supports ordinary view locations and registered namespaces:

```php
view('users.index');
view('blog::posts.index');
```

Namespaced views and namespaced components use the same namespace registry.

## Plain PHP views

A file ending in `.php` is evaluated directly. Hax compilation is applied only to `.hax.php` files.

## Compilation and caching

Hax templates are compiled to PHP and cached. The compiled file is reused until the view cache determines that it has expired.

Compiled Hax files retain the source view path. Rendering errors are wrapped in a view-aware exception that can identify both the original view and its compiled PHP file.

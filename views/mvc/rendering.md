# Rendering Views

Yuga's view layer is built around the view factory, engine, finder, and Hax compiler.

Views are normally stored in `resources/views`. Both `.hax.php` and plain `.php` files are supported.

## Rendering a view

```php
return view('users.profile', ['user' => $user]);
```

Dot, slash, and backslash notation are normalized by the view finder.

## Passing data

Data passed to a view is available in the template:

```php
return view('welcome', ['name' => 'Hamid']);
```

```html
<h1>Hello {{ $name }}</h1>
```

## Includes

```php
@include('partials.header')
```

Includes are rendered as partial views and receive the variables currently defined in the parent template.

## Plain PHP views

A file ending in `.php` is evaluated directly. Hax compilation is applied to `.hax.php` files.

## Compilation and caching

Hax templates are compiled to PHP and cached. The compiled file can then be reused until the source template needs recompilation.

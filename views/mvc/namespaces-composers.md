# View Namespaces, Shared Data, and Composers

## Additional view locations

Applications and packages can add another lookup location:

```php
view()->addLocation($path);
```

The finder searches configured locations in order and supports both `.hax.php` and plain `.php` views.

## Namespaced views

Register a namespace:

```php
view()->addNamespace('blog', __DIR__ . '/views');
```

Then render a namespaced view:

```php
return view('blog::posts.index');
```

The same namespace works with includes:

```php
@include('blog::partials.navigation')
```

and components:

```html
<x-blog::card />
<x-blog::forms.input />
```

A namespaced component is resolved relative to the `components` directory of that namespace.

## Package views

A Composer package can keep its views inside the package:

```text
vendor/acme/yuga-blog/
├── src/
│   └── BlogServiceProvider.php
└── resources/
    └── views/
        ├── posts/
        │   └── index.hax.php
        └── components/
            └── card.hax.php
```

Its service provider can register the package path:

```php
view()->addNamespace(
    'blog',
    dirname(__DIR__) . '/resources/views'
);
```

Because the path is resolved relative to the package, this works normally when Composer installs it under `vendor/`.

## Multiple namespace paths and overrides

Multiple paths may be registered for one namespace:

```php
view()->addNamespace('blog', [
    resources_path('views/vendor/blog'),
    dirname(__DIR__) . '/resources/views',
]);
```

Paths are searched in order, so the application's override can be placed before the package's default views.

`prependNamespace()` can also move a path to the front of an existing namespace.

## Shared data

Values can be shared with every view created by the factory:

```php
view()->share('appName', 'Yuga');

view()->share([
    'company' => 'Acme',
    'year' => date('Y'),
]);
```

Explicit data passed when creating a view overrides shared data with the same key.

## View composers

Composers prepare data when matching views are rendered.

```php
view()->composer('dashboard', function ($view) {
    $view->with('title', 'Dashboard');
});
```

Register the same composer for several views:

```php
view()->composer([
    'dashboard',
    'reports.index',
], $composer);
```

Wildcard patterns are supported:

```php
view()->composer('admin.*', $composer);
view()->composer('*', $composer);
```

The object passed to a composer exposes `name()`, `with()`, and `data()`.

### Class composers

A composer may also be a class name. The class must define a `compose` method:

```php
class NavigationComposer
{
    public function compose($view): void
    {
        $view->with('navigation', buildNavigation());
    }
}

view()->composer('layouts.*', NavigationComposer::class);
```

Composers run immediately before the requested view is resolved and evaluated.

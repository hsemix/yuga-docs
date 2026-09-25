# View Namespaces and Composers

## Additional view locations

Packages and applications can add another lookup location:

```php
view()->addLocation($path);
```

## Namespaced views

Register a namespace:

```php
view()->addNamespace('blog', __DIR__ . '/views');
```

Then render package views with:

```php
view('blog::posts.index');
```

Multiple paths may be registered for one namespace. `prependNamespace()` places a path before existing namespace paths.

## View composers

A composer can prepare data whenever one or more views are rendered:

```php
view()->composer('dashboard', function (array $data) {
    $data['title'] = 'Dashboard';

    return $data;
});
```

## Shared data

For values that should be available to views rendered through the factory:

```php
view()->share('appName', 'Yuga');
```

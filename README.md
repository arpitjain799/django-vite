# Django Vite

[![PyPI version](https://badge.fury.io/py/django-vite.svg)](https://badge.fury.io/py/django-vite)

Integration of [ViteJS](https://vitejs.dev/) in a Django project.

## Installation

### Django

```
pip install django-vite
```

Add `django_vite` to your `INSTALLED_APPS` in your `settings.py`
(before your apps that are using it).

```python
INSTALLED_APPS = [
    ...
    'django_vite',
    ...
]
```

### ViteJS

Follow instructions on [https://vitejs.dev/guide/](https://vitejs.dev/guide/).
And mostly the SSR part.

Then in your ViteJS config file :

- Set the `base` options the same as your `STATIC_URL` Django setting.
- Set the `build.outDir` path to where you want the assets to compiled.
- Set the `build.manifest` options to `true`.
- As you are in SSR and not in SPA, you don't have an `index.html` that
  ViteJS can use to determine which files to compile. You need to tell it
  directly. In your ViteJS config file add the following :

```javascript
export default defineConfig({
  build {
    ...
    rollupOptions: {
      input: {
        <unique key>: '<path to your asset>'
      }
    }
  }
}
```

### Assets

As recommended on Vite's [backend integration guide](https://vitejs.dev/guide/backend-integration.html), your assets should include the modulepreload polyfill.

```javascript
// Add this at the beginning of your app entry.
import 'vite/modulepreload-polyfill';
```

## Usage

### Configuration

- Define a setting variable in your `settings.py` named `DJANGO_VITE_ASSETS_PATH`
  containing the absolute path to where your assets are built.

  - This must correspond to your `build.outDir` in your ViteJS configuration.
  - The `DJANGO_VITE_ASSETS_PATH` must be included in your `STATICFILES_DIRS`
    Django setting.

- Define a setting variable in your `settings.py` named `DJANGO_VITE_DEV_MODE`
  containing a boolean defining if you want to include assets in development
  mode or production mode.

  - In development mode, assets are included as modules using the ViteJS
    webserver. This will enable HMR for your assets.
  - In production mode, assets are included as standard assets
    (no ViteJS webserver and HMR) like default Django static files.
    This means that your assets must be compiled with ViteJS before.
  - This setting may be set as the same value as your `DEBUG` setting in
    Django. But you can do what is good for your needs.

Note : `DJANGO_VITE_ASSETS_PATH` supports `pathlib.Path` syntax or pure `str`.

### Template tags

Include this in your base HTML template file.

```
{% load django_vite %}
```

Then in your `<head>` element add this :

```
{% vite_hmr_client %}
```

- This will add a `<script>` tag to include the ViteJS HMR client.
- This tag will include this script only if `DJANGO_VITE_DEV_MODE` is true,
  otherwise this will do nothing.

Then add this tag (in your `<head>` element too) to load your scripts :

```
{% vite_asset '<path to your asset>' %}
```

This will add a `<script>` tag including your JS/TS script :

- In development mode, all scripts are included as modules.
- In development mode, all scripts are marked as `async` and `defer`.
- You can pass a second argument to this tag to overrides attributes
  passed to the script tag.
- This tag only accept JS/TS, for other type of assets, they must be
  included in the script itself using `import` statements.
- In production mode, the library will read the `manifest.json` file
  generated by ViteJS and import all CSS files dependent of this script
  (before importing the script).
- You can add as many of this tag as you want, for each input you specify
  in your ViteJS configuration file.
- The path must be relative to your `root` key inside your ViteJS config file.
- The path must be a key inside your manifest file `manifest.json` file
  generated by ViteJS.
- In general, this path does not require a `/` at the beginning
  (follow your `manifest.json` file).

```
{% vite_asset_url '<path to your asset>' %}
```

This will generate only the URL to an asset with no tag surrounding it.
**Warning, this does not generate URLs for dependant assets of this one
like the previous tag.**

### Custom attributes

By default, all scripts tags are generated with a `type="module"` and `crossorigin=""` attributes just like ViteJS do by default if you are building a single-page app.
You can override this behavior by adding or overriding this attributes like so :

```
{% vite_asset '<path to your asset>' foo="bar" hello="world" %}
```

This line will add `foo="bar"` and `hello="world"` attributes.

You can also use context variables to fill attributes values :

```
{% vite_asset '<path to your asset>' foo=request.GET.bar %}
```

If you want to overrides default attributes just add them like new attributes :

```
{% vite_asset '<path to your asset>' crossorigin="anonymous" %}
```

Although it's recommended to keep the default `type="module"` attribute as ViteJS build scripts as ES6 modules.

## Vite Legacy Plugin

If you want to consider legacy browsers that don't support ES6 modules loading
you may use [@vitejs/plugin-legacy](https://github.com/vitejs/vite/tree/main/packages/plugin-legacy).
Django Vite supports this plugin. You must add stuff in complement of other script imports in the `<head>` tag.

Just before your `<body>` closing tag add this :

```
{% vite_legacy_polyfills %}
```

This tag will do nothing in development, but in production it will loads the polyfills
generated by ViteJS.

And so next to this tag you need to add another import to all the scripts you have
in the head but the 'legacy' version generated by ViteJS like so :

```
{% vite_legacy_asset '<path to your asset>' %}
```

Like the previous tag, this will do nothing in development but in production,
Django Vite will add a script tag with a `nomodule` attribute for legacy browsers.
The path to your asset must contain de pattern `-legacy` in the file name (ex : `main-legacy.js`).

This tag accepts overriding and adding custom attributes like the default `vite_asset` tag.

## Miscellaneous configuration

You can redefine those variables in your `settings.py` :

- `DJANGO_VITE_DEV_SERVER_PROTOCOL` : ViteJS webserver protocol
  (default : `http`).
- `DJANGO_VITE_DEV_SERVER_HOST` : ViteJS webserver hostname
  (default : `localhost`).
- `DJANGO_VITE_DEV_SERVER_PORT` : ViteJS webserver port
  (default : `3000`)
- `DJANGO_VITE_WS_CLIENT_URL` : ViteJS webserver path to the HMR client used
  in the `vite_hmr_client` tag (default : `@vite/client`).
- `DJANGO_VITE_MANIFEST_PATH` : Absolute path (including filename)
  to your ViteJS manifest file. This file is generated in your
  `DJANGO_VITE_ASSETS_PATH`. But if you are in production (`DEBUG` is false)
  then it is in your `STATIC_ROOT` after you collected your
  [static files](https://docs.djangoproject.com/en/3.1/howto/static-files/) (supports `pathlib.Path` or `str`).
- `DJANGO_VITE_LEGACY_POLYFILLS_MOTIF` : The motif used to find the assets for polyfills inside the `manifest.json` (only if you use [@vitejs/plugin-legacy](https://github.com/vitejs/vite/tree/main/packages/plugin-legacy)).
- `DJANGO_VITE_STATIC_URL_PREFIX` : prefix directory of your static files built by Vite.
  (default : `""`)

  - Use it if you want to avoid conflicts with other static files in your project.
  - It may be used with `STATICFILES_DIRS`.
  - You also need to add this prefix inside vite config's `base`.
    e.g.:

  ```python
  # settings.py
  DJANGO_VITE_STATIC_URL_PREFIX = 'bundler'
  STATICFILES_DIRS = (('bundler', '/srv/app/bundler/dist'),)
  ```

  ```javascript
  // vite.config.js
  export default defineConfig({
    base: '/static/bundler/',
    ...
  })
  ```

## Notes

- In production mode, all generated path are prefixed with the `STATIC_URL`
  setting of Django.

- If you are serving your static files with whitenoise, by default your files compiled by vite will not be considered immutable and a bad cache-control will be set. To fix this you will need to set a custom test like so:

```python
import re

# Vite generates files with 8 hash digits
# http://whitenoise.evans.io/en/stable/django.html#WHITENOISE_IMMUTABLE_FILE_TEST

def immutable_file_test(path, url):
    # Match filename with 12 hex digits before the extension
    # e.g. app.db8f2edc0c8a.js
    return re.match(r"^.+\.[0-9a-f]{8,12}\..+$", url)


WHITENOISE_IMMUTABLE_FILE_TEST = immutable_file_test
```

## Example

If you are struggling on how to setup a project using Django / ViteJS and Django Vite,
I've made an [example project here](https://github.com/MrBin99/django-vite-example).

## Thanks

Thanks to [Evan You](https://github.com/yyx990803) for the ViteJS library.

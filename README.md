//\ Tarp.require - a lightweight JavaScript module loader
=========================================================

Tarp.require is a CommonJS and Node.js compatible module loader licensed as open source under the LGPL v3. It aims to be
as lightweight as possible while not missing any features.

*This is the unstable version of Tarp.require (aka tarp2). Features and interfaces are subject to change and may break
during development!*

## Features

* **Compatible** NodeJS 9.2.0 and CommonJS modules 1.1
* **Plain** No dependencies, no need to compile/bundle/etc the modules
* **Asynchronous** Non-blocking loading of module files
* **Modern** Makes use of promises and other features (support for older browsers via polyfills)
* **Lightweight** Just about 180 lines of code (incl. comments), minified version is about 2kB

### Browser compatibility

* Firefox 29+
* Chrome 33+
* Edge 12+
* Safari 7.1+
* iOS Safari 8+
* Android Browser 4.4.4+
* Opera 20+
* Internet Explorer 10+ (with [URL polyfill](https://github.com/lifaon74/url-polyfill) and [ES6 Promise polyfill](https://github.com/lahmatiy/es6-promise-polyfill))

## Installation

A NPM package only exists for the stable version of Tarp.require, but you can just clone the tarp2-branch of this repository directly or add
it to your git repository as a submodule:

```
$ git submodule add -b tarp2 https://github.com/letorbi/tarp.require.git
```

## Usage

Assuming your domain is *//example.com*, you've installed Tarp.require at */node_modules/@tarp/require* and your
HTML-document is located at *//page/index.html*, you only have to add the following lines to your HTML to load the
script located at */page/scripts/main.js* as your main-module:

```
<script src="/node_modules/@tarp/require/require.min.js"></script>
<script>Tarp.require({main: "./scripts/main.js"});</script>
```

### Enable advanced ID resolving

If you want load IDs without an extension or *index.js* or *package.json* files in directories, you have to enhance your
server configuration a bit. To enable proper ID resolving in the folder */page/node_modules* for nginx, you have to add
the following lines to your sites configuration:

```
location /page/node_modules {
    add_header "Content-Location" "$scheme://$host$uri";
    try_files "$uri" "$uri.js" "${uri}index.js" "${uri}package.json" =404;
}
```

The important value is the *Content-Location* header, which holds the canonical URL for the requested module.

## Synchronous and asynchronous loading

Tarp.require supports synchronous and asynchronous loading of modules. If you load Tarp.require like described above and
use only simple require calls you won't have to care about anything: Just write `require("someModule")` and Tarp.require
will try to load the module and all its sub-modules asynchronously.

However, there might be occasions where you have to force Tarp.require to load a module synchronously. This is possible,
but you should try to avoid this, because synchronous requests might block the loading process of your page and are
therefore [marked as obsolete](https://xhr.spec.whatwg.org/#the-open()-method) by now.

Keep also in mind that synchronous loading of a module prevents the pre-loading of its sub-modules. You have to
explicitly load a sub-module asynchronously to re-start the pre-loading. 

### What modules can be pre-loaded?

Right now only plain require-calls are pre-loaded. This means that the ID of the module has to be one simple string. 
Also require-calls with more than one parameter are ignored.

**Example:** If a module has been loaded asynchronously and contains the require calls `require("Submodule1")`,
`require("Submodule2", true)` and `require("Submodule" + "3")` somewhere in its code, only *Submodule1* will be
pre-loaded, since the require-call for *Submodule2* has more than one parameter and the module-ID in the require-call
for *Submodule3* is not one simple string.

### Enforce synchronous or asynchronous loading

Synchronous or asynchronous loading of a module can be enforced by adding a boolean value as a second
parameter to the require-call:

```
// explicit synchronous loading
var someModule = require("someModule", false);
someModule.someFunc();////

 // explicit asynchronous loading
require("anotherModule", true).then(function(anotherModule) {
    anotherModule.anotherFunc();
});
```

## Path resolving

Inside your main-module (and any sub-module, of course) you can use `require()` as you know it from CommonJS/NodeJS.
Assuming the paths from the "Usage" section and that you are in the main-module, module-IDs will be resolved to the
following paths:

  * `require("someModule.js")` loads *//example.com/page/node_modules/someModule.js*
  * `require("./someModule.js")` loads *//example.com/page/scripts/someModule.js*
  * `require("/someModule.js")` loads *//example.com/someModule.js*

Note that global modules are loaded from */page/node_modules* and not from */node_modules*. This is because the default
global module path is set to `['./node_modules']` and is derived from the location of the page that initializes
Tarp.require.

Without any server configuration Tarp.require is only able to load modules IDs, where the ID matches the exact filename
of the module-script. Tarp.require relies fully on the server to find the correct file for a certain ID. This decision
has been made due to the fact that modules are usually loaded from a remote server and sending multiple request for
different locations would be very time-consuming.

### NPM packages

The loading of a NPM package is the only occasion when Tarp.require might redirect a request on its own. Tarp.require
loads a module-ID specified the `main` field of a *package.json* file, if the following checks are true:

 1. The *package.json* file is loaded indirectly (e.g. via */mypkg* instead of */mypkg/package.json*)
 2. The response contains a valid JSON object 
 3. The object has a property called `main`
 
If this is the case a second request will be triggered to load the modules specified in `main` and the exports of that
module will be returned. Otherwise simply the content of *package.json* is returned.

### The `module.paths` property

Tarp.require always uses the first item of the global `paths` array to resolve an URL from a global module-ID.  Unlike
Node.js it won't iterate over the whole array. Since the global config is always used, any change to `module.paths`
won't affect the loading of modules.

Tarp.require also supports the `require.resolve.paths()` function that returns an array of paths that have been searched
to resolve the given module-ID.

## Options

### Change the global module path

If your modules are not located at *./node_modules/*, you can tell Tarp.require their location via the `paths` option:

```
Tarp.require({main: "./scripts/main", paths: ["/path/to/node/modules"]});
```

### Using `require()` directly in the HTML-document

It is not recommended to load other modules than the main-module directly from your HTML-document. However, if you
really want to use `require()` directly in the HTML-document, you can add `Tarp.require` to the global scope:

```
Tarp.require({expose: true});
```

Keep in mind that a require-call in the HTML-document will load a module synchronously unless you explicitly load it
asynchronously by adding `true` as a second parameter.

### Load the main-module synchronously

If you really need to load the main-module synchronously, you can do it by loading Tarp.require with the
following options:

```
Tarp.require({main: "./scripts/main", sync: true});
```

----

Copyright 2013-2018 Torben Haase \<https://pixelsvsbytes.com>

# WordPress in the browser

[API Reference](https://github.com/WordPress/wordpress-playground/tree/trunk/docs/api/wordpress-playground.md)

This package uses [php-wasm](https://github.com/WordPress/wordpress-playground/blob/trunk/docs/using-php-in-javascript.md) and [php-wasm-browser](https://github.com/WordPress/wordpress-playground/blob/trunk/docs/using-php-in-the-browser.md) to run WordPress fully in the browser and without a PHP server.

The `wordpress-playground` package consists of:

-   Embeddable playground web page
-   WordPress web bundler
-   WordPress-specific setup for the Worker Thread and the Service Worker
-   WordPress-specific automations for tasks like signing in or installing plugins
-   A PHP proxy to download plugins and themes from the WordPress.org directory

## Embeddable playground web page

All parts of this repository come together in the `wordpress.html` page where WordPress is loaded and displayed. 

### Embedding WordPress Playground on other websites

The public WordPress Playground available at [https://wasm.wordpress.net/wordpress.html](https://wasm.wordpress.net/wordpress.html) may be embedded on other websites via the `<iframe>` HTML tag as follows:

```html
<iframe
    style="width: 800px; height: 500px;"
    src="https://wasm.wordpress.net/wordpress.html?mode=seamless"
></iframe>
```

Notice how the URL says `mode=seamless`. This is a configuration option that turns off the "browser UI" and gives WordPress the entire available space.

Here are all the supported configuration options:

* `mode=seamless` or `mode=browser` – Displays WordPress on a full-page or wraps it in a browser UI
* `login=1` – Logs the user in as an admin.
* `page=/wp-admin/` – Load the specified initial page displaying WordPress
* `plugin=coblocks` – Installs the specified plugin. Use the plugin name from the plugins directory URL, e.g. for a URL like `https://wordpress.org/plugins/wp-lazy-loading/`, the plugin name would be `wp-lazy-loading`. You can pre-install multiple plugins by saying `plugin=coblocks&plugin=wp-lazy-loading&…`. Installing a plugin automatically logs the user in as an admin.
* `theme=disco` – Installs the specified theme. Use the theme name from the themes directory URL, e.g. for a URL like `https://wordpress.org/themes/disco/`, the theme name would be `disco`. Installing a theme automatically logs the user in as an admin.
* `rpc=1` – Enables the experimental JavaScript API.

For example, the following code embeds a Playground with a preinstalled Gutenberg plugin, and opens the post editor:

```html
<iframe
    style="width: 800px; height: 500px;"
    src="https://wasm.wordpress.net/wordpress.html?plugin=gutenberg&url=/wp-admin/post-new.php&mode=seamless"
></iframe>
```

### Controlling the embedded WordPress Playground via JavaScript API

**The JavaScript API is an early preview and will likely evolve in the future.**

The embedded Playground can be controlled from JavaScript via `window.postMessage` if you used the `rpc=1` option:

```js
// Ask the Playground whether it has finished booting:
document.querySelector('#playground').contentWindow.postMessage({
   type: 'is_booted',
   requestId: 1
}, '*');

// Receive the messages from Playground:
function handleResponse(e) {
   if(e.data.type === 'response' && e.data.requestId === 1 && e.data.response === true) {
      // Navigate to wp-admin if the Playground was booted:
      document.querySelector('#playground').contentWindow.postMessage({
         type: 'go_to',
         path: '/wp-admin/index.php',
      }, '*');
   }
}
window.addEventListener('message', handleResponse);
```

Playground accepts messages in format `{ "type": <TYPE>, "requestId": <optional Number>, ...data : <optional Object> }` and sends back messages in the same format. At the moment, you need to implement the protocol on your own as there isn't yet JavaScript library to automate that. This page will be updated as soon as one is released.

Playground understands the following messages:

* `{"type": "is_booted", "requestId": <number>}` – Replies with true if the Playground is loaded and ready for messages.
* `{"type": "go_to", "path": <string>}` – Navigates to the requested path.
* `{"type": "rpc", "method": <string>, "args": <string[]>, "requestId": <number>}` – Calls one of the following functions:
  * `run(phpCode: string):` Promise<`{ exitCode: number; stdout: ArrayBuffer; stderr: string[]; }`>
  * `HTTPRequest(request: PHPRequest):` Promise<`{ body: ArrayBuffer; text: string; headers: Record<string, string>; statusCode: number; exitCode: number; rawError: string[]; }`>
  * `readFile(path: string):` Promise<string>
  * `writeFile(path: string, `contents: string): Promise<void>
  * `unlink(path: string):` Promise<void>
  * `mkdirTree(path: string):` Promise<void>
  * `listFiles(path: string):` Promise<string[]>
  * `isDir(path: string):` Promise<boolean>

You will receive the following messages from Playground:

* `{ "type": "response", "requestId": <number>, "data": <any> }` – A response to the message you sent earlier, identified by a unique requestId .
* `{ "type": "new_path", "path": <string> }` – Whenever a new page is loaded in the Playground.


## WordPress web bundler

Web bundler turns a vanilla WordPress release into a browser-optimized one. It:

* Preinstalls the [WordPress SQLite plugin](https://github.com/aaemnnosttv/wp-sqlite-db)
* Runs the WordPress installation wizard
* Reduces the WordPress website size from about 70MB to about 10MB, or 5MB compressed.

Run the bundler with:

```
npm run build:wp
```

The command outputs two files:

* `build/wp.js` – the JavaScript loader for `wp.data`
* `build/wp.data` – the WordPress data bundle consisting of concatenated contents of all WordPress files

Most of the work is done in the [relevant Dockerfile](https://github.com/WordPress/wordpress-playground/blob/trunk/src/wordpress-playground/wordpress/Dockerfile) – consult that file for more details. You can also customize the default WordPress installation by modifying that Dockerfile.

## WordPress-specific setup for the Worker Thread and the Service Worker.

As seen in the [php-wasm-browser package documentation](https://github.com/WordPress/wordpress-playground/blob/trunk/docs/using-php-in-the-browser.md), running PHP in the browser requires a Worker Thread and a Service Worker.

This package provides both, see [worker-thread.ts](https://github.com/WordPress/wordpress-playground/blob/trunk/src/wordpress-playground/worker-thread.ts) and [service-worker.ts](https://github.com/WordPress/wordpress-playground/blob/trunk/src/wordpress-playground/service-worker.ts). The browser expects each to be a separate script so they are not bundled with the main project. Instead, each is its own bundle built into `build/` directory.

WordPress and PHP are both initialized in the Worker Thread. The main application starts the Worker Thread using the `bootWordPress` function:

```js
import { bootWordPress } from './boot';
const workerThread = await bootWordPress();
```

The `bootWordPress` utility downloads the WebAssembly runtime, starts it in a separate thread for performance reasons, and returns a `SpawnedWorkerThread` instance. The main application can control the WordPress instance through the `SpawnedWorkerThread` API.

For example, you can run PHP code using the `run` method:

```js
const result = await workerThread.run(`<?php echo "Hello, world!";`);
console.log(result);
// { stdout: "Hello, world!", stderr: [''], exitCode: 0 }
```

You can also dispatch HTTP requests to WordPress as follows:

```js
const response = await workerThread.HTTPRequest({
	path: workerThread.pathToInternalUrl('/wp-login.php'),
	method: 'GET',
});
console.log(response.statusCode);
// 200
console.log(response.body);
// ... the rendered wp-login.php page ...
```

For more details, see the `SpawnedWorkerThread` reference manual page and the architecture overview.

## WordPress-specific automations for tasks like signing in or installing plugins

### Logging the user in

`wordpress-playground` provides helpers to automate common use-cases, like logging the user in:

```js
import { login } from './macros';

// Authenticate the user by sending a POST request to
// /wp-login.php (via workerThread.HTTPRequest()):
await login(workerThread, 'admin', 'password');
```

### Installing plugins

You can install plugins using the `installPlugin()` helper:

```js
// Login as an admin first
await login(workerThread, 'admin', 'password');

// Download the plugin file
const pluginResponse = await fetch('/my-plugin.zip');
const blob = await pluginResponse.blob();
const pluginFile = new File([blob], 'my-plugin.zip);

// Install the plugin by uploading the zip file to
// /wp-admin/plugin-install.php?tab=upload
await installPlugin(workerThread, pluginFile);
```

## A PHP proxy to download plugins and themes from the WordPress.org directory

The browser cannot simply download a WordPress theme or plugin zip file from the wordpress.org directory because of the cross-origin request policy restrictions. This package provides a [PHP proxy script](https://github.com/WordPress/wordpress-playground/blob/trunk/src/wordpress-playground/plugin-proxy.php) that exposes plugins and themes on the same domain where WordPress Playground is hosted.
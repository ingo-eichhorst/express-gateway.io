---
title: Plugin Development
layout: doc-section
doc-order: 25.2
---

### Background Resources

Many Express Gateway plugins will be built utilizing [Express Middleware][express-middleware] as a starting point.
Express Gateway is built using [ExpressJS][expressjs] and borrows many concepts from it. Building familiarity with it,
especially the concept of [Middleware][express-middleware] will help with understanding Express Gateway plugin
development.

To understand how different entities within a plugin are registered and loaded checkout the
[Express Gateway Boot Sequence]({{ site.baseurl}} {% link docs/runtime/boot-sequence.md %}).

Plugins extend Express Gateway entities as points of extension. These extension points and the plugin framework
development plan are specified within the
[Express Gateway Plugin Specification](https://docs.google.com/document/d/1jSDul2n_xbeKNtnek69M79-geur6aTWShAcBZ9evD0E/edit).

### The Express Gateway Example Plugin

The Express Gateway Plugin Example [npm package][express-gateway-plugin-example-npm] and its
[code on GitHub][express-gateway-plugin-example-github] serves as a guide for how plugins are structured.
The example contains examples of all extension points supported at the time of the
[plugin framework development plan][plugin-development-plan-post].

#### Manually Installing the Example plugin
Express Gateway plugins are automatically istalled using the CLI.  Normally, this example plugin would be installed
using the command `eg plugin install express-gateway-plugin-example`. For the purpose of getting a better understanding
of the mechanics, this section walks through what is normally automated.

[Install][express-gateway-installation] Express Gateway

run the CLI command `eg gateway create` to create Express Gateway instance

```bash
> eg gateway create
? What's the name of your Express Gateway? example-gateway
? Where would you like to install your Express Gateway? example-gateway
? What type of Express Gateway do you want to create? Basic (default pipeline with proxy)
```

- go to the instance folder
- npm install the example package
- edit the `./config/system.config.yml` file and enable the plugin

```bash
cd example-gateway
npm i --save express-gateway-plugin-example
```

Now edit the `./config/system.config.yml` file

Find the following section:

```yml
plugins:
  # express-gateway-plugin-example:
  #   param1: 'param from system.config'
```
Uncomment the `express-gateway-plugin-example` plugin declaration

```yml
plugins:
    express-gateway-plugin-example:
        param1: 'param from system.config'
```

If your configuration is specified in JSON, the equivalent JSON configuration would look like the following:

```json
"plugins": {
    "express-gateway-plugin-example": {
        "param1": "param from system.config"
    }
}
```

#### Running the Example plugin

Run Express Gateway with debugging turned on

```bash
LOG_LEVEL=debug npm start
```

The output provided by the debugging flag should somethig like the following:

```bash
Loading plugins. Plugin engine version: 1.2.0
...
Loaded plugin express-gateway-plugin-example using from package express-gateway-plugin-example
...
registering policy example
...
registering gatewayExtension
...
registering condition url-match
...

```
### Example Plugin Package Overview
The `express-gateway-plugin-example` plugin is an npm package.

Its Main components are:
- `manifest.js` file - contains and exports plugin definition
- `package.json` - contains plugin name and dependencies

All the rest is completely optional. Still, some structure may help. That is why the example plugin contains individual
folders for each extension type

Note: `manifest.js` naming is just a convention used to be more descriptive. The name of this file is configured in
`main` property of `package.json`. Node.js standard naming is index.js.

#### Manifest.js File overview (Plugin Manifest)
An example of the plugin manifest is provided below:

```js
module.exports = {
  version: '1.2.0',
  init: function (pluginContext) {
    // pluginContext.registerX calls
  },
  policies: ['example'],
  schema: {
    param1: {
      type: 'string',
    },
    required: ['param1']
  }
}
```

##### Parameters

- `version` - _optional_ - Hint for the Plugin System how to process plugin, '1.2.0' only at this point
- `init` - _mandatory_ - Function that will be called right after Express Gateway will `require` the plugin package
- `policies` - _optional_ - list of policies to be added to the whitelist (requires confirmation from user)
- `schema` - _optional_ - JSON schema for plugin options. It will be used for prompting during CLI execution and
data validation when loading the plugin.

### JSON Schema support

Plugins, policies and conditions parameters can optionally be validated through a JSON Schema that can be provided
using the `schema` property of the relative part. We suggest to provide such one as it provides a declarative way to
validate the parameters, so you do not have to care about that in your code and can assume everything is ready to be
used.

If not provided, the extension will still be loaded, although a `warn` will be raised.

**Note:** the JSON Schema `$id` property is mandatory, or the Gateway will refuse to load the extension. You can choose
whatever name suits you; however we suggest to use the same convention we have in the gateway, which is:

`http://express-gateway.io/schemas/{extension_type}/{extension_name}.json`

* `extension_type`: Should be `plugin`, `policy` or `condition`
* `extension_name`: It should usually match the extension `name` property.

### Events

Express Gateway exposes several events that plugins can subscribe to to cooordinate loading.

#### Example

```js
module.exports = {
  version: '1.2.0',
  init: function (pluginContext) {
    pluginContext.eventBus.on('hot-reload', function ({ type, newConfig }) {
      // "type" is gateway or system
      // depends on what file was changed
      // newConfig - is newly loaded configuration of ExpressGateway
      console.log('hot-reload', type, newConfig);
    });

    pluginContext.eventBus.on('http-ready', function ({ httpServer }) {
      console.log('http server is ready', httpServer.address());

      // Proxy websockets to localhost:9015
      const httpProxy = require('http-proxy')
      var proxy = new httpProxy.createProxyServer({
        target: {
          host: 'localhost',
          port: 9015
        }
      });

      httpServer.on('upgrade', (req, socket, head) => {
        proxy.ws(req, socket, head);
      });
    });

    pluginContext.eventBus.on('https-ready', function ({ httpsServer }) {
      console.log('https server is ready', httpsServer.address());
    });

    pluginContext.eventBus.on('admin-ready', function ({ adminServer }) {
      console.log('admin server is ready', adminServer.address());
    });
  }
}
```

### Typescript support

Express Gateway is shipped with plugin typings. Therefore, if you're using Typescript to author your plugin or even
Javascript with an appropriate IDE such as [VSCode](https://code.visualstudio.com), you can use those to have type check
as well as intellisense during your development.

#### Typescript

Import the typings and use them. They're put in the global namespace as `ExpressGateway` object.

```javascript
import * as Eg from 'express-gateway';
const plugin : ExpressGateway.Plugin = {…};
```

#### Javascript + VSCode (or another IDE)

- Inform your source file that you'd like to enable typings

```javascript
// @ts-check
/// <reference path="./node_modules/express-gateway/index.d.ts" />
```
- Create a new object and assign it a type using comments

```javascript
/** @type {ExpressGateway.Plugin} */
const plugin = {/*…*/};
```

- Export the plugin as usual

```javascript
module.exports = plugin;
```

For a complete example using this approach, check the
[Rewrite plugin](https://github.com/ExpressGateway/express-gateway-plugin-rewrite)

### Extension Entities Development Guides

The following guides describe how to create Express Gateway entities that can be bundled into a plugin to extend Express
Gateway:

<nav markdown="1">
- [Policy Development guide]({{ site.baseurl}} {% link docs/plugins/policy-development.md %})
- [Condition Development guide]({{ site.baseurl}} {% link docs/plugins/condition-development.md %})
- [Route Development guide]({{ site.baseurl}} {% link docs/plugins/route-development.md %})
</nav>

[express-gateway-installation]: {{ site.baseurl }}{% link docs/installation.md %}
[express-gateway-plugin-example-npm]: https://npmjs.com/express-gateway-plugin-example
[express-gateway-plugin-example-github]: https://github.com/ExpressGateway/express-gateway-plugin-example
[expressjs]: https://www.expressjs.com
[express-middleware]: https://expressjs.com/en/guide/writing-middleware.html
[plugin-development-plan-post]: {{ site.baseurl }}/plugin-development-plan

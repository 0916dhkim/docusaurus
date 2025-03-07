---
sidebar_position: 1
toc_max_heading_level: 4
---

# Lifecycle APIs

During the build, plugins are loaded in parallel to fetch their own contents and render them to routes. Plugins may also configure webpack or post-process the generated files.

## `async loadContent()` {#loadContent}

Plugins should use this lifecycle to fetch from data sources (filesystem, remote API, headless CMS, etc.) or do some server processing. The return value is the content it needs.

For example, this plugin below returns a random integer between 1 and 10 as content.

```js title="docusaurus-plugin/src/index.js"
module.exports = function (context, options) {
  return {
    name: 'docusaurus-plugin',
    // highlight-start
    async loadContent() {
      return 1 + Math.floor(Math.random() * 10);
    },
    // highlight-end
  };
};
```

## `async contentLoaded({content, actions})` {#contentLoaded}

The data that was loaded in `loadContent` will be consumed in `contentLoaded`. It can be rendered to routes, registered as global data, etc.

### `content` {#content}

`contentLoaded` will be called _after_ `loadContent` is done. The return value of `loadContent()` will be passed to `contentLoaded` as `content`.

### `actions` {#actions}

`actions` contain three functions:

#### `addRoute(config: RouteConfig): void` {#addRoute}

Create a route to add to the website.

```ts
type RouteConfig = {
  path: string;
  component: string;
  modules?: RouteModules;
  routes?: RouteConfig[];
  exact?: boolean;
  priority?: number;
};
type RouteModules = {
  [module: string]: Module | RouteModules | RouteModules[];
};
type Module =
  | {
      path: string;
      __import?: boolean;
      query?: ParsedUrlQueryInput;
    }
  | string;
```

#### `createData(name: string, data: any): Promise<string>` {#createData}

A declarative callback to create static data (generally JSON or string) which can later be provided to your routes as props. Takes the file name and data to be stored, and returns the actual data file's path.

For example, this plugin below creates a `/friends` page which displays `Your friends are: Yangshun, Sebastien`:

```jsx title="website/src/components/Friends.js"
import React from 'react';

export default function FriendsComponent({friends}) {
  return <div>Your friends are {friends.join(',')}</div>;
}
```

```js title="docusaurus-friends-plugin/src/index.js"
export default function friendsPlugin(context, options) {
  return {
    name: 'docusaurus-friends-plugin',
    // highlight-start
    async contentLoaded({content, actions}) {
      const {createData, addRoute} = actions;
      // Create friends.json
      const friends = ['Yangshun', 'Sebastien'];
      const friendsJsonPath = await createData(
        'friends.json',
        JSON.stringify(friends),
      );

      // Add the '/friends' routes, and ensure it receives the friends props
      addRoute({
        path: '/friends',
        component: '@site/src/components/Friends.js',
        modules: {
          // propName -> JSON file path
          friends: friendsJsonPath,
        },
        exact: true,
      });
    },
    // highlight-end
  };
}
```

#### `setGlobalData(data: any): void` {#setGlobalData}

This function permits one to create some global plugin data that can be read from any page, including the pages created by other plugins, and your theme layout.

This data becomes accessible to your client-side/theme code through the [`useGlobalData`](../../docusaurus-core.md#useGlobalData) and [`usePluginData`](../../docusaurus-core.md#usePluginData) hooks.

:::caution

Global data is... global: its size affects the loading time of all pages of your site, so try to keep it small. Prefer `createData` and page-specific data whenever possible.

:::

For example, this plugin below creates a `/friends` page which displays `Your friends are: Yangshun, Sebastien`:

```jsx title="website/src/components/Friends.js"
import React from 'react';
import {usePluginData} from '@docusaurus/useGlobalData';

export default function FriendsComponent() {
  const {friends} = usePluginData('docusaurus-friends-plugin');
  return <div>Your friends are {friends.join(',')}</div>;
}
```

```js title="docusaurus-friends-plugin/src/index.js"
export default function friendsPlugin(context, options) {
  return {
    name: 'docusaurus-friends-plugin',
    // highlight-start
    async contentLoaded({content, actions}) {
      const {setGlobalData, addRoute} = actions;
      // Create friends global data
      setGlobalData({friends: ['Yangshun', 'Sebastien']});

      // Add the '/friends' routes
      addRoute({
        path: '/friends',
        component: '@site/src/components/Friends.js',
        exact: true,
      });
    },
    // highlight-end
  };
}
```

## `configureWebpack(config, isServer, utils, content)` {#configureWebpack}

Modifies the internal webpack config. If the return value is a JavaScript object, it will be merged into the final config using [`webpack-merge`](https://github.com/survivejs/webpack-merge). If it is a function, it will be called and receive `config` as the first argument and an `isServer` flag as the second argument.

:::caution

The API of `configureWebpack` will be modified in the future to accept an object (`configureWebpack({config, isServer, utils, content})`)

:::

### `config` {#config}

`configureWebpack` is called with `config` generated according to client/server build. You may treat this as the base config to be merged with.

### `isServer` {#isServer}

`configureWebpack` will be called both in server build and in client build. The server build receives `true` and the client build receives `false` as `isServer`.

### `utils` {#utils}

`configureWebpack` also receives an util object:

- `getStyleLoaders(isServer: boolean, cssOptions: {[key: string]: any}): Loader[]`
- `getJSLoader(isServer: boolean, cacheOptions?: {}): Loader | null`

You may use them to return your webpack configuration conditionally.

For example, this plugin below modify the webpack config to transpile `.foo` files.

```js title="docusaurus-plugin/src/index.js"
module.exports = function (context, options) {
  return {
    name: 'custom-docusaurus-plugin',
    // highlight-start
    configureWebpack(config, isServer, utils) {
      const {getJSLoader} = utils;
      return {
        module: {
          rules: [
            {
              test: /\.foo$/,
              use: [getJSLoader(isServer), 'my-custom-webpack-loader'],
            },
          ],
        },
      };
    },
    // highlight-end
  };
};
```

### `content` {#content-1}

`configureWebpack` will be called both with the content loaded by the plugin.

### Merge strategy {#merge-strategy}

We merge the Webpack configuration parts of plugins into the global Webpack config using [webpack-merge](https://github.com/survivejs/webpack-merge).

It is possible to specify the merge strategy. For example, if you want a webpack rule to be prepended instead of appended:

```js title="docusaurus-plugin/src/index.js"
module.exports = function (context, options) {
  return {
    name: 'custom-docusaurus-plugin',
    configureWebpack(config, isServer, utils) {
      return {
        // highlight-start
        mergeStrategy: {'module.rules': 'prepend'},
        module: {rules: [myRuleToPrepend]},
        // highlight-end
      };
    },
  };
};
```

Read the [webpack-merge strategy doc](https://github.com/survivejs/webpack-merge#merging-with-strategies) for more details.

### Configuring dev server {#configuring-dev-server}

The dev server can be configured through returning a `devServer` field.

```js title="docusaurus-plugin/src/index.js"
module.exports = function (context, options) {
  return {
    name: 'custom-docusaurus-plugin',
    configureWebpack(config, isServer, utils) {
      return {
        // highlight-start
        devServer: {
          open: '/docs', // Opens localhost:3000/docs instead of localhost:3000/
        },
        // highlight-end
      };
    },
  };
};
```

## `configurePostCss(options)` {#configurePostCss}

Modifies [`postcssOptions` of `postcss-loader`](https://webpack.js.org/loaders/postcss-loader/#postcssoptions) during the generation of the client bundle.

Should return the mutated `postcssOptions`.

By default, `postcssOptions` looks like this:

```js
const postcssOptions = {
  ident: 'postcss',
  plugins: [require('autoprefixer')],
};
```

Example:

```js title="docusaurus-plugin/src/index.js"
module.exports = function (context, options) {
  return {
    name: 'docusaurus-plugin',
    // highlight-start
    configurePostCss(postcssOptions) {
      // Appends new PostCSS plugin.
      postcssOptions.plugins.push(require('postcss-import'));
      return postcssOptions;
    },
    // highlight-end
  };
};
```

## `postBuild(props)` {#postBuild}

Called when a (production) build finishes.

```ts
interface Props {
  siteDir: string;
  generatedFilesDir: string;
  siteConfig: DocusaurusConfig;
  outDir: string;
  baseUrl: string;
  headTags: string;
  preBodyTags: string;
  postBodyTags: string;
  routesPaths: string[];
  plugins: Plugin<any>[];
  content: Content;
}
```

Example:

```js title="docusaurus-plugin/src/index.js"
module.exports = function (context, options) {
  return {
    name: 'docusaurus-plugin',
    // highlight-start
    async postBuild({siteConfig = {}, routesPaths = [], outDir}) {
      // Print out to console all the rendered routes.
      routesPaths.map((route) => {
        console.log(route);
      });
    },
    // highlight-end
  };
};
```

## `injectHtmlTags({content})` {#injectHtmlTags}

Inject head and/or body HTML tags to Docusaurus generated HTML.

`injectHtmlTags` will be called both with the content loaded by the plugin.

```ts
function injectHtmlTags(): {
  headTags?: HtmlTags;
  preBodyTags?: HtmlTags;
  postBodyTags?: HtmlTags;
};

type HtmlTags = string | HtmlTagObject | (string | HtmlTagObject)[];

type HtmlTagObject = {
  /**
   * Attributes of the HTML tag
   * E.g. `{'disabled': true, 'value': 'demo', 'rel': 'preconnect'}`
   */
  attributes?: {
    [attributeName: string]: string | boolean;
  };
  /**
   * The tag name e.g. `div`, `script`, `link`, `meta`
   */
  tagName: string;
  /**
   * The inner HTML
   */
  innerHTML?: string;
};
```

Example:

```js title="docusaurus-plugin/src/index.js"
module.exports = function (context, options) {
  return {
    name: 'docusaurus-plugin',
    loadContent: async () => {
      return {remoteHeadTags: await fetchHeadTagsFromAPI()};
    },
    // highlight-start
    injectHtmlTags({content}) {
      return {
        headTags: [
          {
            tagName: 'link',
            attributes: {
              rel: 'preconnect',
              href: 'https://www.github.com',
            },
          },
          ...content.remoteHeadTags,
        ],
        preBodyTags: [
          {
            tagName: 'script',
            attributes: {
              charset: 'utf-8',
              src: '/noflash.js',
            },
          },
        ],
        postBodyTags: [`<div> This is post body </div>`],
      };
    },
    // highlight-end
  };
};
```

Tags will be added as follows:

- `headTags` will be inserted before the closing `</head>` tag after scripts added by config.
- `preBodyTags` will be inserted after the opening `<body>` tag before any child elements.
- `postBodyTags` will be inserted before the closing `</body>` tag after all child elements.

## `getClientModules()` {#getClientModules}

Returns an array of paths to the [client modules](../../advanced/client.md#client-modules) that are to be imported into the client bundle.

As an example, to make your theme load a `customCss` or `customJs` file path from `options` passed in by the user:

```js title="my-theme/src/index.js"
const path = require('path');

module.exports = function (context, options) {
  const {customCss, customJs} = options || {};
  return {
    name: 'name-of-my-theme',
    // highlight-start
    getClientModules() {
      return [customCss, customJs];
    },
    // highlight-end
  };
};
```

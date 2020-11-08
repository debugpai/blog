## Aliasing Preact correctly for Server Side Rendering


React has become the most modern front end library. However there is a pretty big disadvantage of using it. `react` and `react-dom` library together make up 38.6kB minified and gzipped (version 16.13.1). This is quite a significant cost to pay on mobile devices on slow internet connections. [This article by Addy Osmani](https://v8.dev/blog/cost-of-javascript-2019) does a good job in highlighting the cost of javascript. [Preact](https://preactjs.com/) is a fast 3kB alternative to React with the same modern API. It makes sense to use Preact if you want to optimize for first load time. However if you have a heavy application with a lot of user interactions, then react is more optimized.

Preact is lighter mostly because it doesn't ship [synthetic events](https://reactjs.org/docs/events.html) that are bundled into `react-dom`. It tries to be as close to browser features as possible. [These are all the differences](https://preactjs.com/guide/v10/differences-to-react) between react and preact.

## Aliasing
Aliasing with `preact/compat` is a good way to migrate your react app to start using preact without changing the source code. All you have to do is add [a new alias](https://webpack.js.org/configuration/resolve/#resolvealias) like this to your webpack config

```
// webpack.config.js

const webpackConfig = { 
  ...
  "resolve": { 
    "alias": { 
      "react": "preact/compat",
      "react-dom/test-utils": "preact/test-utils",
      "react-dom": "preact/compat",
      // Must be below test-utils
    }
  }
}
```

During bundling, whenever webpack encounters a file with `import ... from 'react'`, it swaps react to `preact/compat`. So `import React from 'react'` becomes `import React from 'preact/compat'` in the generated bundles. Note that this substitution happens not only for your code but also for the libraries that you're using. So for instance if you're using `react-redux`, all `react` imports in the library are swapped to use `preact`.

If you're using another bundler you can check the official docs [here](https://preactjs.com/guide/v10/getting-started#aliasing-react-to-preact)

### Server side rendering
The above mentioned technique works well with client but you might face some problems if you're using server side rendering. This is especially true if you have any kind of an [externals](https://webpack.js.org/configuration/externals/) setup in your webpack config. Externals allows for specifying specific libraries that you don't want included in your generated bundles. Webpack assumes that these would be available at runtime. 

Server side rendered applications usually have a different bundling flow for the server than for the client. Thus different bundle(s) is generated for the server than the client. With server side rendered applications, it is a common practice to use [node externals](https://github.com/liady/webpack-node-externals) which tells webpack not to include bundles in node_modules for the server bundles.
These are replaced with `require` statements which are then resolved by node's module resolution system at runtime.

This brings us a problem if we're aliasing with `preact/compat`. While aliasing client side modules, we're able to swap `react` with `preact/compat` for all dependencies. But when these modules are externalized, webpack no longer go through the dependencies and thus doesn't alias `preact/compat`. Whenever it encounters a dependency, it says its not my responsibility, let node handle it.

And node has no idea that it needs to swap `react` with `preact/compat`. We need to tell it how to correctly resolve `react`.

#### Enter `module-alias`
[module-alias](https://github.com/ilearnio/module-alias) is a handy package that allows us to tell node to use `preact/compat`. It hooks into node's module resolution system and changes it according to our needs. We need to add this code to our application

```
const moduleAlias = require('module-alias');
moduleAlias.addAlias('react', 'preact/compat/dist/compat.js');
moduleAlias.addAlias('react-dom', 'preact/compat/dist/compat.js');
```

Note that `moduleAlias` has to be added with `require` statement and not an `import`. This is because `import`s are not resolved synchronously. 
And we want `moduleAlias` to be executed synchronously before the rest of the code that references the aliased modules.

Also note that this has to be at the very beginning of your server execution for it to work.
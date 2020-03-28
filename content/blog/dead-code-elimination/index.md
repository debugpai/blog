---
title: Dead code elimination in Javascript
description: 
---

Dead code elimination is a process wherein code that is not used is excluded from the code that is executed. In many compile time languages this is a much easier process since a compiler can easily determine the code that is used. However in javascript, this can be quite tricky since this clarity is not present.

There are several forms of dead code in javascript
- Unused source files
- Environment specific code
- Unused code in same file 

## Unused source files
In the early days of javascript this used to be a real problem since unused source files had to be manually cleaned up. But with modern bundlers this becomes a really easy process. If a file is not imported anywhere then it is automatically excluded from the resulting bundle. There are also some plugins like [this one](https://www.npmjs.com/package/unused-files-webpack-plugin) which also help us to remove these from the codebase.

## Environment specific code
In a real world app we have several environment specific code. These can be production, staging and development, mobile vs desktop or even domain specific logic. So when we build an app for production, we don't want to serve code to the user which is only used during development. 

We can do this by using [define-plugin](https://webpack.js.org/plugins/define-plugin/) in webpack or [plugin-replace](https://github.com/rollup/plugins/tree/master/packages/replace) in rollup. 

```js
// webpack.config.js
new webpack.DefinePlugin({
  'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV)
});
```
(webpack 4 does this automatically. this is just for showing what happens under the hood)

```js
// rollup.config.js
plugins: [
  replace({ "process.env.NODE_ENV": JSON.stringify(process.env.NODE_ENV) })
]
```

What this does is if you have code that looks like the following
```js
let API_HOST;
if (process.env.NODE_ENV === 'production') {
    API_HOST = 'production-host.mycompany.com';
} else {
    API_HOST = 'localhost:9000';
}
```

Given `NODE_ENV=production`, it converts it to the following
```js
if ('production' === 'production') {
    API_HOST = 'production-host.mycompany.com';
} else {
    API_HOST = 'localhost:9000';
}
```

Then the minifier removes the unreachable code, reducing it down to the following,
```
API_HOST = 'production-host.mycompany.com';
```

### Caveat
This only works if you don't reassign the variable defined in the bundler config. In this case `process.env.NODE_ENV`.

So if we change the code like the following
```js
let API_HOST;
const IS_PROD = process.env.NODE_ENV === 'production';
if (IS_PROD) {
    API_HOST = 'production-host.mycompany.com';
} else {
    API_HOST = 'localhost:9000';
}
```
Then the bundler will not be able to eliminate the dead code because it cannot know for sure the correct value of `IS_PROD` inside the if statement. Since the value could have changed after assignment of the variable and the if statement.

If you do want to use `IS_PROD` as a variable and have dead code elimination available then you can declare it in the plugin config like this:

```js
// webpack.config.js
new webpack.DefinePlugin({
  'IS_PROD': process.env.NODE_ENV === 'production'
});
```

```js
// rollup.config.js
plugins: [
  replace({ 'IS_PROD': process.env.NODE_ENV === 'production' })
]
```

## Unused code in same file 
There are 2 kinds of unused code that can be part of the same file.
- Unused exports
- Side effects

This is the most difficult type of code to dead code eliminate because Javascript is not a compiled language. However modern bundlers are finding smart ways to do it. They use a technique called as [tree shaking](https://webpack.js.org/guides/tree-shaking/) where they rely on static structure of ES Modules to achieve it.

### Unused Exports
In modern web development, we use various libraries to get the job done. The problem however is that we don't need all the code that the library ships. 

Consider a file like the following
```js
// math.js
function add (x, y) {
    return x + y;
}

function subtract (x, y) {
    return x - y;
}
```

If we have another file where we import only one of the functions like
```js
import { subtract } from './math';

console.log(subtract(3, 2));
```

Then in the final resulting bundle only the `subtract` function is included. Add is excluded.

We can also combine this with Environment specific code which was highlighted in the previous section.

```js
import { subtract } from './math';

if (process.env.NODE_ENV === 'production') {
    console.log('This is production');
} else {
    console.log('Subtraction in dev mode only ' + subtract(1, 3));
}
```

Here we are calling `subtract` function only in development. In cases like this, `subtract` is bundled only in development mode. In production mode, neither `add` nor `subtract` functions are bundled.

This is possible because ES modules are static in nature as opposed to `commonjs` modules which due to their dynamic nature makes it impossible for the bundler to determine which modules to tree shake.
```js
if (process.env.NODE_ENV === 'production') {
    require('./math.js').add(1, 2);
} else {
    require('./math.js').subtract(2, 3)
}
```

This is the main reason why [lodash](https://lodash.com/) which was based on commonjs modules created a new package [lodash-es](https://github.com/lodash/lodash/tree/es) which is based on ES modules. This means that if we use only 1 function from `lodash-es` only that function will be bundled into our main bundle.

```js
import { reverse } from 'lodash-es';

console.log(reverse([1, 2, 3]));
```

In the above example only the `reverse` function will be bundled into the code.

### Side Effects












## References
- [Google blog reduce js payload with tree shaking](https://developers.google.com/web/fundamentals/performance/optimizing-javascript/tree-shaking)
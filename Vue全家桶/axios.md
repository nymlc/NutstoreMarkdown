首先看`pkg.json`，从中可知入口文件为`index.js`

```js
// index.js
module.exports = require('./lib/axios');
```

而其内实际引入的是`lib/axios.js`，我们以此逐步浏览

# lib/axios

这个就是主入口，我们先回顾下`axios`用法

```js
axios({
    method: 'post',
    url: '/user/12345',
    data: {
        firstName: 'Fred',
        lastName: 'Flintstone'
    }
})
```

可见`axios`必然是个函数，我们再看此入口文件的`export`

```js
'use strict';

var utils = require('./utils');
var bind = require('./helpers/bind');
var Axios = require('./core/Axios');
var mergeConfig = require('./core/mergeConfig');
var defaults = require('./defaults');

/**
 * Create an instance of Axios
 *
 * @param {Object} defaultConfig The default config for the instance
 * @return {Axios} A new instance of Axios
 */
function createInstance(defaultConfig) {
    var context = new Axios(defaultConfig);
    var instance = bind(Axios.prototype.request, context);

    // Copy axios.prototype to instance
    utils.extend(instance, Axios.prototype, context);

    // Copy context to instance
    utils.extend(instance, context);

    return instance;
}

// Create the default instance to be exported
var axios = createInstance(defaults);

// ...

module.exports = axios;

// Allow use of default import syntax in TypeScript
module.exports.default = axios;
```

它是调用`createInstance`的返回值，这其实就是**工厂模式**，它的返回值`instance`可见就是`Axios.prototype.request`，也就是我们调用的`axios`其实就是它



 

众所周知，`JS`是需要被宿主环境解析运行的，就像运行在浏览器。但是环境总是各异，不同的版本对`ES`的支持程度不尽相同，索性目前市面上的浏览器（环境）基本上都支持`ES5`，这样子我们就可以把`ES6`们转成`ES5`不就行了
当然，我们可以自己写`polyfill`，但是很明显这不靠谱，也费劲，小范围还好，项目大了就不卡普，况且说来说去就是在原型链上增改，源代码里充斥着各种`polyfill`，一旦遇到之后新版本的浏览器可能会出现不可控的问题，这时候我们就需要一个转换工具，这就是`babel`
# ES


| 版本 | 发表日期   | 与前版本的差异                                               |
| ---- | ---------- | ------------------------------------------------------------ |
| 1    | 1997年6月  | 首版                                                         |
| 2    | 1998年6月  | 格式修正，以使得其形式与ISO/IEC16262国际标准一致             |
| 3    | 1999年12月 | 强大的正则表达式，更好的词法作用域链处理，新的控制指令，异常处理，错误定义更加明确，数据输出的格式化及其它改变 |
| 4    | 放弃  | 由于关于语言的复杂性出现分歧，第4版本被放弃，其中的部分成为了第5版本及Harmony的基础 |
| 5    | 2009年12月  | 新增“严格模式（strict mode）”，一个子集用作提供更彻底的错误检查,以避免结构出错。澄清了许多第3版本的模糊规范，并适应了与规范不一致的真实世界实现的行为。增加了部分新功能，如getters及setters，支持JSON以及在对象属性上更完整的反射 |
| 5.1    | 2011年6月  | ECMAScript标5.1版形式上完全一致于国际标准ISO/IEC 16262:2011。 |
| 6    | 2015年6月  | ECMAScript 2015（ES2015），第 6 版，最早被称作是 ECMAScript 6（ES6），添加了类和模块的语法，其他特性包括迭代器，Python风格的生成器和生成器表达式，箭头函数，二进制数据，静态类型数组，集合（maps，sets 和 weak maps），promise，reflection 和proxies。作为最早的 ECMAScript Harmony 版本，也被叫做ES6 Harmony。 |
| 7    | 2016年6月  | ECMAScript 2016（ES2016），第 7 版，多个新的概念和语言特性 |
| 8    | 2017年6月  | ECMAScript 2017（ES2017），第 8 版，多个新的概念和语言特性 |
| 9    | 2018年6月  | ECMAScript 2018 （ES2018），第 9 版，包含了异步循环，生成器，新的正则表达式特性和 rest/spread 语法。 |
| 10    | 2019年6月  | ECMAScript 2019 （ES2019），第 10 版 |
| 11    | 2020年6月  | ECMAScript 2020 （ES2020），第 11 版 |

该表格来自于维基百科，在`ES6`之前版本，命名就如第一列所示，但是到`ES6`有了变化，因为`ES6`和`ES5`变化实在是太大了，更是有各方的多次提交，在一个版本里属实不现实，那么应该也是`ES6.x`，但是标准委员会觉得应该将版本更新形成一种常规流程，各个组织、个人可以提案，每隔一段固定时间就商讨下接受哪些提案，并且每年6月份进行发版，命名就按照年份来，`ES6`是`2015/6`，所以也就是`ES2015`
其实至目前为止`ES2015`之后的版本都是对其的小幅增改，所以也可以泛指它们为`ES6`


# 工作原理

可能我们用的很多，但是知其然而不知其所以然是不对的

## 用法

一般而言就三种用法：
+ `babel-standalone`单体文件在线转换
+ `babel-cli`命令行
+ `webpack`等构建工具的`babel-loader`类似的插件

我们一般而言都是用后两个，其实应该是最后一个，第二个一般都是本地运行调试所用

## 运行方式

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/TB1nP2ONpXXXXb_XpXXXXXXXXXX-1958-812.png" alt="img" style="zoom: 50%;" />

[此图](https://developer.aliyun.com/article/62671)可以一览其运行方式，就三点：解析、转换、生成

这其实可以参见`Vue`的模板解析那一套

+ 解析

  就是把代码解析成`AST`，`JS`引擎都是有自己的解析器，`babel`的解析器是`babylon`，解析过程就和`Vue`解析模板差不多，词法分析、语法分析就是了

+ 转换

  这里就是`babel`插件介入的地方，我们通过`babel-traverse`深度优先遍历`AST`，对节点进行增删改

+ 生成

  这里就是把魔改后的`AST`重新生成（`babel-generator`）代码字符串就是了（`new Function(code)`）

一般而言前两步骤是插件介入的地方

## 插件

其实通过上面流程可知**转换**操作是插件干的，它本身并没有转换功能。还有就是**解析**也有介入，这就把插件分类了：

+ 语法插件

  这是作用在解析阶段，是辅助解析器解析的插件，就像`@babel/plugin-syntax-dynamic-import`，辅助解析`import()`。也就是有些写法是非法的，没有插件处理会报错，所以需要插件介入

+ 转译插件

  这个就很明显就是转换代码，就像箭头函数转成普通函数

> 如果同一类语法有语法、转译插件俩个版本，那么用转译一个即可

## 配置（`.babelrc`）

配置文件就是`.babelrc`，基本格式如下

```json
{
  "presets":[],
  "plugins":[]
}
```

`package.json`文件里的`babel`字段也可以，格式俩者一致，一般而言有这俩个就够了，不过还有其它几个捎带提下

### ignore

```json
{
  	"ignore": ["./a.js"]
}
```

就是在此列的文件不转译

### minified

压缩文件用的，不过一般来说没啥用，就像`webpack`已经有`UglifyJsPlugin`了

### comments

就是去掉注释，`webpack`也有了

### env

就是不同的环境不同的配置

```json
{
    "env": {
        // test 是提前设置的环境变量，如果没有设置BABEL_ENV则使用NODE_ENV，如果都没有设置默认就是development
        "test": {
            "presets": ["env", "stage-2"]
        }
    }
}
```

## Preset（预设）

插件很多，动辄十几个，我们一个个配置估计会崩溃，所以官方贴心的整了个预设，就像是套餐

+ 官方`Preset`

  + `@babel/preset-env`，最常用的，它通过配置得知目标环境特点，然后只针对目标环境进行转换，如此加快构建速度，减小转换后代码体积

  + `@babel/preset-flow`，适用于`flow`

  + `@babel/preset-react`，适用于`react`

  + `@babel/preset-typescript`，适用于`ts`

+ `Stage-X` （实验性质的 Presets）

  + Stage 0 - 设想（Strawman）：只是一个想法，可能有 Babel插件。

  + Stage 1 - 建议（Proposal）：这是值得跟进的。

  + Stage 2 - 草案（Draft）：初始规范。

  + Stage 3 - 候选（Candidate）：完成规范并在浏览器上初步实现。

  + Stage 4 - 完成（Finished）：将添加到下一个年度版本发布中。

    低级的`Stage`会包括高级的`Stage`，就像[`babel-preset-stage-0`](https://sourcegraph.com/github.com/babel/babel@v6.6.3/-/blob/packages/babel-preset-stage-0/index.js)包括[`babel-preset-stage-1`](https://sourcegraph.com/github.com/babel/babel@v6.6.3/-/blob/packages/babel-preset-stage-1/index.js)，而且没有`Stage-4`，因为它会在下一年更新时被放进`env`
  
+ `es20xx`

  这是已经纳入规范的了，



### [env](https://babeljs.io/docs/en/babel-preset-env#targets)

这个最常用，它的原理就是通过配置确定目标环境然后做对应的转换，所以不配置的话就是相当于全给转换了
#### 运行机制

其实[源码](https://github.com/babel/babel/tree/main/packages/babel-preset-env)看来是不难的
1. 生成`babel`插件与环境的关系
```js
yarn build-data
```
这就会生成俩个文件：`plugins.json、built-ins.json`，前者是插件和环境对应关系，后者是一些内置方法比如`Array.prototype.includes`和环境的对应关系

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/image-20201010154018483.png" alt="image-20201010154018483" style="zoom:50%;" />

2. 解析配置`targets`

```json
{
    "presets": [
        [
            "env",
            {
                "targets": {
                    "browsers": [
                        "last 2 versions",
                        "safari >= 7"
                    ]
                }
            }
        ]
    ]
}
```

配置也简单，如上所示适配浏览器最新的俩个版本且`safari`版本大于等于`7.0`，此例以`targets`为例

通过`browserslist`来解析`.babelrc`之类的配置来获取到目标环境情况，然后和上一步的`plugins.json、built-ins.json`相结合自然可以得出该环境需要哪些插件了

> `browserslist`可以参见[这篇译文](https://juejin.im/post/6844903669524086797#heading-13)
>
> 值得注意的是其`JS API`使用如下所示
>
> ```js
> import browserslist from "browserslist";
> browserslist(targets.browsers) // { chrome: '54.0.0', node: '6.10.0', ie: '10.0.0' }
> ```
>
> 

#### 配置

还有其它更多的配置项，**以7.11.0为例**，可以参见[官方文档](https://babeljs.io/docs/en/babel-preset-env)，可以[在线转换](https://babeljs.io/repl#?browsers=Edge%2016&build=&builtIns=false&spec=false&loose=false&code_lz=Q&debug=false&forceAllTransforms=false&shippedProposals=false&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=unambiguous&lineWrap=true&presets=env%2Cenv&prettier=true&targets=&version=7.11.6&externalPlugins=)

+ targets: `string | Array<string> | { [string]: string }`, defaults to `{}`
  
  顾名思义，它就是用来确定目标环境的
  
  ```json
  // browserslist-compatible查询语句
  {
    	"targets": "> 0.25%, not dead"
  }
  // 支持的最小版本
  // chrome, opera, edge, firefox, safari, ie, ios, android, node, electron
  {
      "targets": {
        	"chrome": "58",
        	"ie": "11"
      }
  }
  ```
  
  + esmodules: `boolean`
  
    浏览器可能支持[ES Modules](https://www.ecma-international.org/ecma-262/6.0/#sec-modules)，若是配置了该项，那么将会忽略目标环境。它实际上会自己使用一套支持`ES Module`的最低版本配置，如下
  
    ```json
    {
        chrome: '61',
        and_chr: '61',
        edge: '16',
        firefox: '60',
        and_ff: '60',
        node: '13.2.0',
        opera: '48',
        op_mob: '48',
        safari: '10.1',
        ios_saf: '10.3',
        samsung: '8.2',
        android: '61'
    }
    ```
  
  + node: `string | "current" | true`
  
    针对`node`的配置项，`current、true`相当于`"node": process.versions.node`，也就是针对当前`node`版本
  
  + safari: `string | "tp"`
  
    `"safari": tp`就是针对`Safari`的技术预览版`technology preview`
  
  + browsers: `string | Array<string>`
  
    即将被废除的一个配置项
  
+ bugfixes: `boolean`

  可以看这个[链接](https://babeljs.io/blog/2020/03/16/7.9.0)，大致意思就是转换插件不是一出来就是没bug的，就像`@babel/plugin-transform-function-parameters`，在转换的时候（`edge 16`）会出现原本不用转的却转成`ES5`代码，[如此例](https://babeljs.io/repl#?browsers=Edge%2016&build=&builtIns=false&spec=false&loose=false&code_lz=PTAEFEBMHMFNQIwDZQAsCGBnU7QCMBXaUAdzAAd0AnTASwDtjNUB7KgFw3slElk3ZUCAY3YEqsSACgQoSlXQBbWO1g1SYPgDN0BADbtQAN3R6C_UrU4McVKixKgtBeqNot6mKcI8CnLFlAAXlAACgBvHGDEUABfABp8aIAmRIA6DOpoTABKYIA-UABtdES8RKzMAF0AbiA&debug=false&forceAllTransforms=false&shippedProposals=false&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=unambiguous&lineWrap=true&presets=env%2Cenv&prettier=true&targets=&version=7.9.0&externalPlugins=)，开启了该配置项就可以解决这个问题

+ spec: 'boolean'

  开启的话就会给预设里的插件列表启用它们更加严格的符合规范的要求，不过这样子一来编译就更慢了

  就像转换箭头函数，开启的话还会检测它的`this`指向是否正确，不开启就只是简单转换成普通函数

  ```js
  // 未开启
  "use strict";
  
  var fn = function fn() {};
  
  // 开启
  "use strict";
  
  var _this = void 0;
  
  function _newArrowCheck(innerThis, boundThis) { 
      if (innerThis !== boundThis) { 
          throw new TypeError("Cannot instantiate an arrow function"); 
      } 
  }
  
  var fn = function fn() {
      _newArrowCheck(this, _this);
  }.bind(void 0);
  ```

+ loose: `boolean`

  就是懒散模式，开启的话会生成更贴近`ES5`的代码，兼容性更好，不过自然就更远离`ES next`了

  自然是不开启生成更严格遵循`ES next`语法的更安全，所以一般不建议开启

+ modules: `"amd" | "umd" | "systemjs" | "commonjs" | "cjs" | "auto" | false`

  就是转换模块语法，`cjs`就是`commonjs`简写，打包出来的文件就是指定的模式

+ debug: `boolean`

  输出编译目标环境、使用的插件、`plugin data`版本等，就是个开发模式

  <img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/2020%252010%252012%252013%252058%252041%2520.png" alt="image-20201012135612253" style="zoom:50%;" />

+ include: `Array<string|RegExp>`

  插件数组，在此配置里的插件即使目标环境不需要也会被使用

  有效选项包括：

  + `Babel`插件：支持全写和简写，`@babel/plugin-transform-spread、plugin-transform-spread`
  + 内置对象（`built-ins`）：`es6.set`

  可接受的输入：

  + 全写：`string: "es6.math.sign"`
  + 部分：`string: "es6.math.*"`，类似于通配符
  + 正则：`/^transform-.*/`

  > 这个插件在目标环境原生实现有`bug`的时候就很有用了，没设置的话可能就不会拍平这个缺陷，设置了就没问题了
  >
  > `include、exclude`只能作用于[预设内的插件](https://github.com/babel/babel/blob/master/packages/babel-compat-data/scripts/data/plugin-features.js)，外的插件就和`preset-env`平级设置即可

+ exclude:  `Array<string|RegExp>`

  和`include`类似，黑名单而已

+ useBuiltIns: `"usage"` | `"entry"` | `false`

  该选项配置`preset-env`如何处理`polyfill`

  1. `useBuiltIns: 'entry'`

     要是需要根据不同的基于环境的入口引入不同的`core-js`，这选项就会用个新插件来替换`import "core-js/stable"`和`import "regenerator-runtime/runtime"`声明(或者`require("corejs")`和`require("regenerator-runtime/runtime")`)

     **其实就是把`import '@babel/polyfilll'`根据配置的`targets`转换为具体的引入语句，当然不包含浏览器已经支持的部分，也就是不管你用没用到，只要目标环境不支持的都会引入对应的`polyfill`**
     ```js
     // in
     import "core-js";
     // out
     import "core-js/modules/es.string.pad-start";
     import "core-js/modules/es.string.pad-end";
     ```

     > 整个项目里只能有一次`import "core-js"`和`import "regenerator-runtime/runtime"`，多次引用会报错。`@babel/polyfill`已经包含了`core-js`

  2. `useBuiltIns: 'usage'`

     这个比`entry`智能在它只引入这个项目里用过的不支持的特性

     ```js
     // in
     var a = new Promise();
     
     // out（若是浏览器不支持）
     import "core-js/modules/es.promise";
     var a = new Promise();
     
     // out（若是浏览器支持）
     var a = new Promise();
     ```
     
  3. `useBuiltIns: false`
  
     就是不作处理
  
+ corejs: `2, 3`或者`{ version: 2 | 3, proposals: boolean }`

  配合`useBuiltIns`使用，只有其值是`entry、usage`时才会生效，这是确保引入了正确的`core-js`版本

+ forceAllTransforms: `boolean`

  强制所有转换，我们可以结合[Javascipt config file](https://www.babeljs.cn/docs/config-files#javascript)，它可以获取到环境变量，这样子可以根据不同的环境来设置是否全部转换，一般来说智能转化才好

+ configPath: `string`，默认`process.cwd()`

  搜索`Browserslist`配置的起点，以此起点向上搜索至系统根目录

+ ignoreBrowserslistConfig: `boolean`

  [browserslist config sources](https://github.com/browserslist/browserslist#queries)，它有多处配置所在，`Browserslist`配置文件（`.browserslistrc`文件）、`pkg`文件里的`browserslist`选项，这选项开启就会忽略配置

  > 在`Browserslist`配置文件不用于`babel`的时候有用
  >
  > 俩个配置同时存在会报错

+ browserslistEnv: `string`

  设置环境，[参见此即可](https://github.com/browserslist/browserslist#configuring-for-different-environments)，就是可以为不同的环境指定不同的浏览器查询语句

+ shippedProposals: `boolean`

  开启的话就不会对浏览器某些提案的原生实现的语法进行转换

## plugins 与 presets 的执行顺序

配置文件里同时存在`presets、plugins`就会存在执行先后，先执行`plugins`，再执行`presets`

`plugins`顺序执行，`presets`逆序执行。逆序是为了向下兼容，我们写`presets`一般都是安装时间顺序列出，这样子一来先执行新的，若是先执行旧的会出错

# 常用工具

| 名称                                          | 说明                                              | 备注                                                     |
| --------------------------------------------- | ------------------------------------------------- | -------------------------------------------------------- |
| babel-cli                                     | 命令行编译转译文件                                |                                                          |
| babel-node                                    | 命令行工具，执行脚本                              | Babel 6.x时是babel-cli的一部分，Babel 7.x独立于babel-cli |
| babel-register                                | 改写了`require`命令，其加载的文件会被转译         |                                                          |
| babel-polyfill                                | 为所有的API添加兼容方法                           | 需要在代码执行前就加载，体积庞大                         |
| babel-plugin-transform-runtime  babel-runtime | 工具方法每次替换不用重复，改用`require`，精简代码 | babel-runtime需要安装为依赖，不是开发依赖                |

## babel-cli vs babel-node

这俩个在`Babel 6.x`的时候`babel-node`还只是`babel-cli`的一部分，但是到了`Babel 7.x`，`@babel/node`就从`@babel/cli`独立出来

前者只是转译文件，后者执行文件

## babel-register

它就是改写了`require`命令，这样子被`require`的脚本都会被转译，也就可以实时运行了

## babel-plugin-transform-runtime

这就是精简代码的，babel转换代码会有很多的`helpers`通用函数，就像是转译`class`

```js
// in
class Person {
    
}
// out without babel-plugin-transform-runtime
"use strict";

function _classCallCheck(instance, Constructor) {
    if (!(instance instanceof Constructor)) {
        throw new TypeError("Cannot call a class as a function");
    }
}


var Person = function Person() {
    _classCallCheck(this, Person);
};
// out with babel-plugin-transform-runtime
"use strict";

var _interopRequireDefault = require("@babel/runtime/helpers/interopRequireDefault");

var _classCallCheck2 = _interopRequireDefault(require("@babel/runtime/helpers/classCallCheck"));


var Person = function Person() {
    (0, _classCallCheck2.default)(this, Person);
};
```

很明显看出来它的效用，就是不用每个文件都插入这段类似`_classCallCheck`之类的通用函数，**这也是为什么`@babel/runtime`是必要依赖的缘由**

## @babel/polyfill

### Syntax(句法)与API

`Babel`把`JS`语法分为句法和`API`
+ 句法就是类似`let、const、class`之类的无法重写的部分
+ `API`就是`Promise、Array.from`之类的可以重写的，它分为俩部分
    + 可以直接调用的方法，`Promise、Object.keys`
    + 实例方法，`[].includes`

`Babel`只会转化句法，`API`就要另做她想了

### core-js & regenerator-runtime/runtime

转译`API`还是简单的，就是加垫片`polyfill`
```js
// in
var a = new Promise();
[1, 2, 3].includes(1);

async function fn() {

}
// out
"use strict";

require("core-js/modules/es.array.includes");

require("core-js/modules/es.object.to-string");

require("core-js/modules/es.promise");

require("regenerator-runtime/runtime");

function asyncGeneratorStep(gen, resolve, reject, _next, _throw, key, arg) { try { var info = gen[key](arg); var value = info.value; } catch (error) { reject(error); return; } if (info.done) { resolve(value); } else { Promise.resolve(value).then(_next, _throw); } }

function _asyncToGenerator(fn) { return function () { var self = this, args = arguments; return new Promise(function (resolve, reject) { var gen = fn.apply(self, args); function _next(value) { asyncGeneratorStep(gen, resolve, reject, _next, _throw, "next", value); } function _throw(err) { asyncGeneratorStep(gen, resolve, reject, _next, _throw, "throw", err); } _next(undefined); }); }; }

var a = new Promise();
[1, 2, 3].includes(1);

function fn() {
  return _fn.apply(this, arguments);
}

function _fn() {
  _fn = _asyncToGenerator( /*#__PURE__*/regeneratorRuntime.mark(function _callee() {
    return regeneratorRuntime.wrap(function _callee$(_context) {
      while (1) {
        switch (_context.prev = _context.next) {
          case 0:
          case "end":
            return _context.stop();
        }
      }
    }, _callee);
  }));
  return _fn.apply(this, arguments);
}
```
可见这俩个库都被引入了，`@babel/polyfill`就集成了这两个

#### core-js

这个库包含了`ES5、ES6+`等几乎所有的`polyfill`实现，它也可以被单独引用

#### regenerator-runtime/runtime

来自于`facebook`的一个[库](https://github.com/facebook/regenerator)，实现了`generator/yeild、async/await`

## @babel/runtime

我们知道`@babel/polyfill`是加垫片，但是它有个很严重的问题，假设我们开发一个类库给别人用，那么这个类库就有`Promise`的实现，但是使用这个类库的人自己也有`Promise`的实现，这样子一来就会被覆盖，显然这是不合理的
这时候我们可以选择另外一个法子，那就是`API`替换，借助`@babel/plugin-transform-runtime`即可
```json
// 配置
{
    "plugins": [
        [
            "@babel/transform-runtime",
            {
                "corejs": 3
            }
        ]
    ]
}
```
```js
// in
var a = new Promise();
[1, 2, 3].includes(1);
// out
"use strict";

var _interopRequireDefault = require("@babel/runtime-corejs3/helpers/interopRequireDefault");

var _includes = _interopRequireDefault(require("@babel/runtime-corejs3/core-js-stable/instance/includes"));

var _promise = _interopRequireDefault(require("@babel/runtime-corejs3/core-js-stable/promise"));

var _context;

var a = new _promise.default();
(0, _includes.default)(_context = [1, 2, 3]).call(_context, 1);
```
可见无论是`Promise`还是`.includes`都被替换了，不会污染全局
> 必须得用`corejs3.x`，`corejs2.x`还不行，其实`corejs3.x`在极端情况下也是不行的
> 比如我们在调用`[].includes(1)`这样语句，`babel`通过监测到`includes`关键词自然可以进行转译，但是若如下就不行了
>
> ```js
> // in
> const arr = Math.random(1) <= 0.5 ? [1, 2, 3] : []
> const key = Math.random(1) <= 0.5 ? 'includes' : 'push'
> arr[key](1)
> // out
> "use strict";
> 
> var arr = Math.random(1) <= 0.5 ? [1, 2, 3] : [];
> var key = Math.random(1) <= 0.5 ? 'includes' : 'push';
> arr[key](1);
> ```

# Babel 7.x

有了很大的变化，不过核心不变，具体可看[官网](https://babeljs.io/docs/en/v7-migration)，下面列几个常见的，要是想从`Babel 6.x`升级到`Babel 7.x`，可以考虑`babel-upgrade`

## preset

`stage-x、es20xx`这俩个都没了，不推荐使用了，推荐使用`env`，这样子开发者就不用多加考虑了

## 包名

`babel-*`变成`@babel/*`，就像`babel-preset-env`变成`@babel/preset-env`，`preset、plugin`可以省略，所以也可以是`@babel/env`

`babylon`也变成了`@babel/parser`

一些插件包名也改了，就不一一列举了

## @babel/node

`@babel/node`独立于`@babel/cli`，不像`Babel 6.x`，`babel-node`是`babel-cli`的一部分

# Babel插件自实现

我们依照文头的那张流程图一一道来，[可见文档](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md)
## 解析

首先就是把代码解析成`AST`，这里可以使用[在线工具](https://astexplorer.net/)来解析，至于`AST`的有什么类型可以[参见于此](https://github.com/babel/babel/blob/master/packages/babel-parser/ast/spec.md#binaryexpression)
`Babel`解析是引用了`babylon`这个库，当然在`Babel 7.x`就改为`@babel/parser`
```js
import { parse } from "@babel/parser";
parse(code, parserOpts);
```
用法如上所示，`parserOpts`是解析的配置，可以参见官网

## 转化

`AST`得到了就得去遍历改造它，我们自己遍历的话肯定就麻烦了，还好有现成的库`@babel/traverse`

### visitor

```js
import * as parser from "@babel/parser";
import traverse from "@babel/traverse";

const code = `function square(n) {
  return n * n;
}`;

const ast = parser.parse(code);

traverse(ast, {
    enter(path) {
        if (path.isIdentifier({ name: "n" })) {
            path.node.name = "x";
        }
    }
});
```
可见`traverse`的配置对象是关键，**转换**也是在这里面实现，但是转换是插件实现的，所以这个对象就得是插件提供，这个对象我们称为`visitor`

## 生成

得到改造之后的`AST`就可以使用`@babel/generator`将其生成为代码字符串即可
```js
import {parse} from '@babel/parser';
import generate from '@babel/generator';

const code = 'class Example {}';
const ast = parse(code);

const output = generate(ast, { /* options */ }, code);
```

## 插件实现

自上可知对于插件而言就`@babel/traverse`是最关键的，而我们要关心的就是`visitor`对象的构建
[可见文档](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md)，有俩个插件得注意下：

+ @babel/types
    这个就是一个处理`AST`的工具库，它包含了构造、验证、变换`AST`节点的方法，比如判断下这个节点是否是数值表达式（`isNumericLiteral`）
+ @babel/template
    这个就是个用来生成复杂`AST`的，通过字符串模板可以很轻松的生成一个`AST`
    ```js
    import template from "@babel/template";
    import generate from "@babel/generator";
    import * as t from "@babel/types";
    
    const buildRequire = template(`
    var IMPORT_NAME = require(SOURCE);
    `);
    
    const ast = buildRequire({
    IMPORT_NAME: t.identifier("myModule"),
    SOURCE: t.stringLiteral("my-module")
    });
    
    console.log(generate(ast).code); // var myModule = require("my-module");
    ```
    简单的实现个插件
```js
/**
 * const num = 1 + 2 * 3 / 2
 * 这样子的话可以直接编译为   var num = 4;
 * 
*/
const t = require('@babel/types')

function replaceAST(path) {
    const { node: { left, right, operator } } = path
    let result
    if (t.isNumericLiteral(left) && t.isNumericLiteral(right)) {
        switch(operator) {
            case '+':
                result = left.value + right.value
                break
            case '-':
                result = left.value - right.value
                break
            case '*':
                result = left.value * right.value
                break
            case '/':
                result = left.value / right.value
                break
        }
        if (result !== undefined) {
            path.replaceWith(t.numericLiteral(result))
        }
    }
}

const visitor = {
    BinaryExpression: {
        enter(path) {
            replaceAST(path)
        },
        exit(path) {
            replaceAST(path)
        }
    }
}

module.exports = function() {
    return {
        visitor
    }
}
```


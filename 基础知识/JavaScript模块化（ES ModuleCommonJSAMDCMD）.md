# 1模块化历史
## 1.1前言
参照[前端模块化开发的价值](https://github.com/seajs/seajs/issues/547)
## 1.2无模块化
每次说到JavaScript都会想到**Brendan Eich**花了十来天就发明了它，那就是JS的鸿蒙时期，混沌初开。
就像当年在校初学前端时写的代码，没有那么多的套路，就是从上往下码代码，没有想着去声明函数神马的，甚至多少代码都写在一个JS里。现在想来真是惨不忍睹。虽然本人入坑前端距那个鸿蒙时代实在久远，但是据各种典籍记载，那时候的代码风格就差不多这样子，从上往下一直堆着就好了。
```javascript
var a = 0;
if (xxx) {
  // 省略100L
}
document.getElementById('id').onclick = function(event) {
  // 省略若干行
}
......
```
## 1.3模块化冒泡
每个行业都有梗，现在和同事聊天有时候还会吐槽十几年前的老网站，真是有幸见过。前辈的聊天更有意思了，当年的登录居然是写死在前端代码里，就像这样子
```javascript
if (username === 'xxxxx' && password === 'xxxxxx') {
  // 登录成功
}
```
是不是觉得很无语。当年的前端都是静态页面，没有现在这样子通过ajax和后端交互神马的，内容更是丰富多彩，更新及时。
前端代码愈发庞大，那么自然而然会暴露很多问题。
无非就俩个：

* 命名冲突
* 文件依赖
### 1.3.1命名冲突解决
1.  java风格的namespace，这个很好理解，在此不赘诉，缺点的话，自行想象，不堪
2.  自执行函数(内部变量不可见不被污染)
```javascript
// jQuery式的匿名自执行函数
// 缺点是增添了全局变量、依赖需要提前提供
(function(root) {
	root.jQuery = window.$ = jQuery; // 挂载到window之上
})(window)
```

```javascript
// 普通自执行函数
// 缺点就是暴露了全局变量，而且随着模块的增加，全局变量会很多
module = function() {
	function module() {

	}
	return module;
}()
```
3. *YUI3的沙箱机制*，这个表示不晓得。
### 1.3.2文件依赖解决
这个没有解决方法，乖乖自行保证顺序和不缺漏吧
```javascript
<script src="https://cdn.bootcss.com/underscore.js/1.8.3/underscore-min.js"></script>
<script src="https://cdn.bootcss.com/backbone.js/1.3.3/backbone-min.js"></script>
```
## 1.4CommonJS
### 1.4.1前言
随着前端的发展，到node.js被创，js可以用来写server端代码。
做过后端的同学肯定知道没有模块化怎么能忍呢？
我们通过上节所得，我们可以得出以下几点需待解决
* 模块代码安全，不可被污染也不可污染别人，沙箱呀
* 把模块接口暴露出去（得优雅呀，不能增添全局变量）
* 这个依赖顺序管理
### 1.4.2发展
这个还真没有经历了解过。度娘一番，幸好看到[seajs下的一个issues](https://github.com/seajs/seajs/issues/588)。
大致就是大牛很牛，推出了[Modules/1.0规范](http://wiki.commonjs.org/wiki/Modules)。
之后为了推广到浏览器端，大牛产生分歧，分为三大流派：
* Modules/1.x 流派（[[Modules/Transport](http://wiki.commonjs.org/wiki/Modules/Transport)
](http://wiki.commonjs.org/wiki/Modules/Transport)通过工具转换现有的CommonJS）
* Modules/Async 流派（自立门派）
* Modules/2.0 流（[[Modules/Wrappings](http://wiki.commonjs.org/wiki/Modules/Wrappings)
](http://wiki.commonjs.org/wiki/Modules/Wrappings)对1.0的升级）

**这里说下为什么不能用在浏览器**
* 服务端代码在硬盘，加载模块时间几乎忽略不计。浏览器端就不成了。
* 模块引用未被function，所以暴露在了全局之下。
### 1.4.3番外（AMD、CMD）
[AMD](https://github.com/amdjs/amdjs-api/wiki/AMD-(%E4%B8%AD%E6%96%87%E7%89%88))是 RequireJS 在推广过程中对模块定义的规范化产出。
[CMD](https://github.com/seajs/seajs/issues/242)是 SeaJS 在推广过程中对模块定义的规范化产出。
## 1.5ES Module
这个是ECMA搞得一套。和之前的区别在于人家是官方的，根正苗红，上文的是社区搞得，野生。
# 2 ES Module/CommonJS/AMD/CMD差异
## 2.1 ES Module与CommonJS的差异
### 编译时和运行时
首先说下编译时和运行时。JavaScript有俩种声明方法（声明变量和声明方法）。var/const/let和function
编译时，对于声明变量会在内存中开辟一块内存空间并指向变量名，且指向变量名，赋值为undefined。对于函数声明会一样的开启空间。不过赋值为声明的函数体。PS：无论顺序如何，都会先声明变量
运行时，执行变量初始化之类的。
```javascript
// 源码
var a = 3;
function f() {
	return 'f';
}

// 编译时
var a = undefined;
var f = function() {
	return 'f';
}
// 运行时
a = 3;
```
CommonJS模块是对象，是运行时加载，运行时才把模块挂载在exports之上（加载整个模块的所有），加载模块其实就是查找对象属性。
ES Module不是对象，是使用export显示指定输出，再通过import输入。此法为编译时加载，编译时遇到import就会生成一个只读引用。等到运行时就会根据此引用去被加载的模块取值。所以不会加载模块所有方法，仅取所需。
* CommonJS 模块输出的是一个值的拷贝，ES6 模块输出的是值的引用。
* CommonJS 模块是运行时加载，ES6 模块是编译时输出接口。
[详情参见](http://es6.ruanyifeng.com/#docs/module-loader#ES6-%E6%A8%A1%E5%9D%97%E4%B8%8E-CommonJS-%E6%A8%A1%E5%9D%97%E7%9A%84%E5%B7%AE%E5%BC%82)
## 2.2CommonJS与AMD/CMD的差异 
AMD/CMD是CommonJS在浏览器端的解决方案。CommonJS是同步加载（代码在本地，加载时间基本等于硬盘读取时间）。AMD/CMD是异步加载（浏览器必须这么干，代码在服务端）
## 2.3AMD与CMD的差异 
* AMD是提前执行（RequireJS2.0开始支持延迟执行，不过只是支持写法，实际上还是会提前执行），CMD是延迟执行
* AMD推荐依赖前置，CMD推荐依赖就近
# 3 用法
## 3.1 CommonJS
```javascript
// 导出使用module.exports，也可以exports。exports指向module.exports；即exports = module.exports
// 就是在此对象上挂属性
// commonjs
module.exports.add = function add(params) {
    return ++params;
}
exports.sub = function sub(params) {
    return --params;
}

// 加载模块使用require('xxx')。相对、绝对路径均可。默认引用js，可以不写.js后缀
// index.js
var common = require('./commonjs');
console.log(common.sub(1));
console.log(common.add(1));
```
## 3.2 AMD/RequireJS
* 定义模块：**define(id?, dependencies?, factory)**
* 加载模块：**require([module], factory)**
```javascript
// a.js
// 依赖有三个默认的，即"require", "exports", "module"。顺序个数均可视情况
// 如果忽略则factory默认此三个传入参数
// id一般是不传的，默认是文件名
define(["b", "require", "exports"], function(b, require, exports) {
	console.log("a.js执行");
	console.log(b);
// 暴露api可以使用exports、module.exports、return
	exports.a = function() {
		return require("b");
	}
})
// b.js
define(function() {
	console.log('b.js执行');
	console.log(require);
	console.log(exports);
	console.log(module);
	return 'b';
})
// index.js
// 支持Modules/Wrappings写法，注意dependencies得是空的，且factory参数不可空
define(function(require, exports, module) {
    console.log('index.js执行');
    var a = require('a');
    var b = require('b');
})
// index.js
require(['a', 'b'], function(a, b) {
	console.log('index.js执行');
})
```
## 3.3 CMD/SeaJS
SeaJS平时没有到，不过了解了下，丰富用法看[CMD定义规范](https://github.com/seajs/seajs/issues/242)。
* 定义模块：define(factory)
```javascript
// a.js
// require, exports, module参数顺序不可乱
// 暴露api方法可以使用exports、module.exports、return
// 与requirejs不同的是，若是未暴露，则返回{}，requirejs返回undefined
define(function(require, exports, module) {
	console.log('a.js执行');
	console.log(require);
	console.log(exports);
	console.log(module);
})
// b.js
// 
define(function(require, module, exports) {
	console.log('b.js执行');
	console.log(require);
	console.log(exports);
	console.log(module);
})
// index.js
define(function(require) {
	var a = require('a');
	var b = require('b');
	console.log(a);
	console.log(b);
})
```
定义模块无需列依赖，它会调用factory的toString方法对其进行正则匹配以此分析依赖。**预先下载，延迟执行**
## 3.4 ES Module 
### 输出/export
```javascript
// 报错1
export 1;
// 报错2
const m = 1;
export m;

// 接口名与模块内部变量之间，建立了一一对应的关系
// 写法1
export const m = 1;
// 写法2
const m = 1；
export { m };
// 写法3
const m = 1；
export { m as module };
```
**PS：这个有点不是很明白，大致理解就是不能直接导出变量，但是可以导出声明（函数、变量声明）。这里的接口理解是export之后的变量，它和变量建立了映射关系。总的而言，export之后只能接声明或者语句**
### 输入/import
#### 基本用法
```javascript
// 类似于对象解构
// module.js
export const m = 1;
// index.js
// 注意，这里的m得和被加载的模块输出的接口名对应
import { m } from './module';
// 若是想为输入的变量取名
import { m as m1 }  './module';
// 值得注意的是，import是编译阶段，所以不能动态加载，比如下面写法是错误的。因为'a' + 'b'在运行阶段才能取到值，运行阶段在编译阶段之后
import { 'a' + 'b' } from './module';
// 若是只是想运行被加载的模块，如下
// 值得注意的是，即使加载两次也只是运行一次
import './module';
// 整体加载
import * as module from './module';
```
PS：CommonJS和ES Module是可以写一起的，但是最好不要。毕竟一个是编译阶段一个是运行阶段。就在项目中入过坑，自行体会。
#### 赋值
首先输入的模块变量是不可重新赋值的，它只是个可读引用，不过却可以改写属性
```javascript
// 单例
// module.js
export const a = {};
// module2.js
export { a } from './module';
import { a as a1 } from './module';
import { a } from './module2';
a1.e = 3;
console.log(a1) // { e: 3 }
console.log(a) // { e: 3 }
```
### 输出/export default
```javascript
// module.js
// 其实export default就是export { xxx as default }
const m = 1；
export default m;
<===>
export { m as default }
// index.js
// 对应的输入也得相应变化
import module from './module';
<===>
import { default as module } from './module';
```
还记得之前export小结处的俩报错么？如下写法正确，因为提供了default接口
```javascript
// 写法1
export default 1;
// 写法2
const m = 1;
export default m;
// 错误写法
export default const m = 1;
```
PS：export default只能一次
### [复合写法](http://es6.ruanyifeng.com/#docs/module#export-%E4%B8%8E-import-%E7%9A%84%E5%A4%8D%E5%90%88%E5%86%99%E6%B3%95)
可用于模块间继承。比如在a模块写下如下，那么a模块不就有了./module的方法了
```javascript
export { a } from './module';
export { a as a1 } from './module';
export * from './module';
```
## 动态加载/import()
因为编译时加载，所以不能动态加载模块。不过幸好有import()方法。
这家伙返回值是一个Promise对象，所以then、catch你开心就好
```javascript
// 普通写法
import('./module').then(({ a }) => {})
// async、await
const { a } = await import('./module');
```
# 4 番外自实现
```javascript
var MyModules = (function(){
	var modules = [];
	function define(name, deps, cb) {
		deps.forEach(function(dep, i) {
			deps[i] = modules[dep];
		});
		modules[name] = cb.apply(cb, deps);
	}
	function get(name) {
		return modules[name];
	}
	return {
		define: define,
		get: get
	};
})();
MyModules.define('add', [], function() {
	return function(a, b) {
			return a + b;
		};
})
MyModules.define('foo', ['add'], function(add) {
	var a = 3;
	var b = 4;
	return {
		doSomething: function() {
			return add(a, b) + a;;
		}
	};
})
var add = MyModules.get('add');
var foo = MyModules.get('foo');
console.log(add(1, 2));
console.log(foo.doSomething());
```
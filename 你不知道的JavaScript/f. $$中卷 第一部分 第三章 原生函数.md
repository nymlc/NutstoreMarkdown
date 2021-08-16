# 内部属性[[Class]]
只要是typeof返回值为'object'的对象都包含一个[[Class]]内部属性。它是一个内部的分类，和传统意义上的类不能一概而论。我们无法直接访问这个属性，一般通过```Object.prototype.toString```来查看
一般而言它与创建这个对象的内建函数相对应（在ES6上可以自定义）
```javascript
Array.prototype[Symbol.toStringTag] = "Foo"
Reflect.toString.call([]) // [object Foo]
```
即使null、undefined这样子的没有构造函数的基本类型值也能返回```[object Null]、[object Undefined]```
# 封装对象包装
基本类型值没有.length、.toString之类的属性和方法，我们能访问是因为JS自动为我们包装了这个基本类型值
#### 封装对象释疑
+ 封装对象就是对象，肯定是真值
+ 可以使用Object()来封装基本类型值（可以不带new）
+ 基本类型值始终只是基本类型值
```javascript
'a' instanceof String // false
'a'.__proto__ === String.prototype // true
```
# 拆封
既然有封装那么就有拆封，我们可以用```valueOf()```（Object.prototype上），此为显示拆封
若是在运算之类用到基本类型值，那么就会隐式拆封
```javascript
var a = new String('a')
typeof (a + '') // "string"
typeof a // "object"
```
# 原生函数作为构造函数
我们喜欢使用字面量来创建Array、Object、Function、RegExp等等。其实和构造调用效果是一样的。**创建的值都是通过封装对象来包装**
1. Array(...)
```javascript
// new可带可不带，不带的话会自行补上
// 参数只有一个时会当做数组的预设长度，有点类似C的开辟数组单元。但是JS可没有这玩意，它会创建一个稀疏数组
var a = new Array(1, 2, 3)
var b = [1, 3, 4]
```
> 稀疏数组（包含至少一个空单元），可以使用delete创造空单元（显示的值是undefined，不过却和显示设置成undefined不一样）、通过设置.length来创造空单元、通过new Array()来创造空单元

> 一个没有数组单元，不过length却有值得数组通常表现在类数组上，感觉完全是arguments之类的类数组当初设定导致的，可被废止
空单元和显示设置为undefined的数组很不一样
+ 在各大浏览器包括它们的不同版本控制台输出也不一样
+ 操作也不尽相同
```javascript
var a = new Array(3)
var b = [undefined, undefined, undefined]
a.join() // ",,"
b.join() // ",,"
a.map((itm, i) => i) // [empty × 3](chrome76.0.3809.100)
b.map((itm, i) => i) // [0, 1, 2]
```
首先map表现不一样好理解，因为a完全没有任何单元，所以就没有循环
join表现一样是因为其内部实现，大致可参考下方
```javascript
function fakeJoin(arr, connector) {
    var str = '';
    for (var i = 0; i < arr.length; i++) {
        if (i > 0) {
            str += connector;
        }
        if (arr[i] !== undefined) {
            str += arr[i];
        }
    }
    return str;
}
```
其实感觉可以整一章各种polyfill大合集
+ 我们可以通过```var a = Array.apply( null, { length: 3 } )```来创建**类空数组**
其实这个是利用了apply第二参数可传类数组这个特性。这里其实是相当于```var a = Array.apply( null, [undefined, undefined, undefined] )```
2. Object(..)、Function(..) 和 RegExp(..)
+ 其实主要是Function(..)，这玩意的构造用法可能很多人都不会用
```javascript
var foo = new Function('a', 'return "Foo: "+ a')
foo(2) // "Foo: 2"
// 相当于
function foo(a) {
    return "Foo: " + a
}
```
+ Object(..)还是偶尔有用的，例如得到封装对象等等。其实字面量写法很好用了，构造方法不能一次性设置多个属性
+ RegExp(..)还是很好用的，可以动态设置正则表达式，比如动态匹配一些字符串等等
```javascript
new RegExp('\d+', 'g')
/\d+/g
```
3. Date(..)和Error(..)
这俩个很重要，因为它们没有字面量。
+ Date()和new Date()有差，前者返回日期字符串值，后者返回日期对象。参数是Unix时间戳（1970/1/1起，秒为单位）
日期对象的.getTime方法可获得时间戳，ES5的Date.now()方法也可以获取当前时间戳
+ Error(..)带不带new都可以，和Array类似。创建Error对象主要是为了获取到当前运行栈的上下文。此对象拥有俩属性：stack、message。大多数JS引擎都是通过只读属性stack来访问运行栈上下文，message是创建Error对象时传入的参
4. Symbol(..)
Symbol（符号）也是个内置函数，由它创建的自定义符号是基本数据类型。自定义符号只能```Symbol(..)```，不能带new关键字，否则会报错
若是一个对象有以符号为属性的，```Object.getOwnPropertyNames()```是取不到该属性名的，可以使用```Object.getOwnPropertySymbols()```
很多人喜欢用**带_前缀**的属性来命名私有属性，现在可以用符号来命名了。就像```for
..in```之类的遍历不得
5. 原生原型
原生函数的原型自然也是对象，不过它并非普通对象那么简单。就像``Function.prototype``是空函数，``RegExp.prototype``是空正则，``Array.prototype``是空数组。
所以我们可以将原型作为默认值，这是个很好的默认值
```javascript
function Foo(vals = Array.prototype, fn = Function.prototype, rx = RegExp.prototype) {

}
```
采用这种赋值的好处就是.prototype已经被创建且仅创建一次了，若是使用``[]、function () {}``之类的，那么每次调用都会创建（其实也是看具体的JS引擎，之后可能会被垃圾回收），造成内存和CPU的浪费（其实基本可忽略）
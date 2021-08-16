# \_\_proto\_\_
除了ES6的Proxy，均是之前说过的[[Get]]、[[Put]]
1. Object.prototype：所有普通（内置，不是特定主机的扩展）的\_\_proto\_\_的尽头
2. 属性的设置和屏蔽**（属性设置完整过程）**：
```javascript
obj.foo = 'bar'

// 1. 如果obj对象上包含名为foo的属性，那么只会修改已有的属性值
// 2. 如果foo不在obj上，那么\_\_proto\_\_会被遍历
// 3. 如果\_\_proto\_\_上面找不到，那么foo会被添加到obj上
// 4. 如果\_\_proto\_\_上面找到了，那么就复杂了。

// ps：总会取原型链底层的属性

// 如果foo不直接存在obj上，而是在其原型链上，该赋值会出现以下三种情况
// 1. 如果foo被标记为writable: true（writable默认true），那么会直接在obj上添加foo这个新属性，它是屏蔽属性
// 2. 如果被标记为writable: false，那么无法修改已有的属性，或者创建屏蔽属性。在严格模式下还会抛出错误（TypeError）
// 3. 如果存在setter，那么会调用这个setter
// 值得注意的是，2、3可以使用Object.defineProperty来越过（可以屏蔽属性）。也就是=操作符才会符合上面俩情况
"use strict";
var o = {}
Object.defineProperty(o, 'foo', {
	value: 'bar',
	writable: false
})
var obj = Object.create(o)
obj.foo = 123
```
> 第二种情况主要是模拟类属性继承。比如o是父类，obj是子类。o的属性foo是只读，它被obj继承，自然也是只读，所以obj无法创建foo。这个限制只在=操作符上
# "类"
原型链使得一个对象关联到另外一个对象，但是为什么要这样子做呢？
1. 类函数
PS：这里讲一个误区。一说起原型就会想到函数对象的``prototype``属性，其实这只是函数对象的一个不可枚举的属性，它的指向的对象才是原型
```javascript
function Foo() {

}
var foo = new Foo()

Object.getPrototypeOf(foo) === Foo.prototype // true
```
在``new Foo()``时，会创建foo对象，这个对象的\_\_proto\_\_会被指向``Foo.prototype``
回忆一下上一章所说的。若是面向类的语言之中，类的实例化其实是把类的行为复制到物理对象里（可操作的），每一个新实例都是这样子。
不过在JS里没有类的复制，只能创建多个对象，这些对象都通过\_\_proto\_\_关联到``Foo.prototype``。
也就是我们并没有实例化一个类（因为我们没有从一个类复制任何行为到一个对象），只是关联了俩对象。**值得注意的是，这里说的类很明显指的是Foo.prototype，呼应上一章**
>其实上面所说的就是原型继承，不过这个名字很明显有问题（JS里就没有继承这回事）。其实正确的叫法应该是委托。毕竟对象本身上没有的行为通过**关联**，可以在另外一个对象上找到。

>对象之间的关系不是复制而是委托

**差异继承**：描述对象行为时，只描述其与不同于普遍的特征（就像描述白种人就说他们金发碧眼，而不会说他们俩只眼睛）。其实还是委托比较符合
```javascript
Object.prototype.clone = function() {
    var o = Object.create(this)
    return o
}

var root = {}

var person = root.clone()
person.name = 'person'
person.toString = function() {
    return this.name
}
var oli = person.clone()
oli.name = 'oli'

onload.toString() // oli
```
2. 构造函数
主要是几点让我们觉得JS里有类。一个是``new``，一个是``Foo()``有点像构造函数调用，一个是``Foo.prototype.constructor``这个对象（.constructor指向的是对象关联的函数，值得注意的是构造出来的对象也能引用这个属性，不过这个对象本身没有这个属性）
> 在JS中没有构造函数，当且仅当使用new是，函数调用会变成构造函数调用
3. 技术
上面说道``constructor``是在``.prototype``上的。这个其实很坑。可以被修改。**o.constructor**是一个非常不靠谱的引用
# （原型）继承
![原型继承](https://upload-images.jianshu.io/upload_images/4874009-8e263e3c87b0665a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```javascript
function Foo(name) {
    this.name = name
}
Foo.prototype.myName = function () {
    return this.name
}

function Bar(name, label) {
    Foo.call(this, name)
    this.label = label
}
// 这句是核心，指代的是上图的那条线
// ES5语法，会抛弃默认的prototype，比如constructor被破坏了
Bar.prototype = Object.create(Foo.prototype)
// ES6方法，可以不用重写Bar.prototype
Object.setPrototypeOf(Bar.prototype, Foo.prototype)
// 俩点误区
// Bar.prototype = Foo.prototype，这个肯定不行，因为扩展子类行为会影响到父类
// Bar.prototype = new Foo()，这个倒是没有上面这个问题，不过有个hen严重的问题，调用了Foo，那么Foo函数会被莫名调用一次

Bar.prototype.myLabel = function () {
    return this.label
}

var bar = new Bar('bar', 'child')

console.log(bar.myName(), bar.myLabel()) // bar child
```
1. 检查”类“关系
寻找一个对象的委托对象就像在Java里检查一个实例的继承祖先（反射），有俩种方法
```javascript
function Bar() {

}
Bar.prototype.myLabel = function () {
    
}

var bar = new Bar()
```
+ ``bar instanceof Bar``：这个操作符的意思就是在**左值**的整条原型链上是否有**右值**``.prototype``指向的对象。很明显，这只能处理对象（左值）和函数对象（右值）的关系。
```
var a = ''
a.__proto__ === String.prototype // true ①
a instanceof String // false ②
// 这点很奇怪，按理a的原型链上的确出现了String.prototype，不过instanceof却是false
// 这里其实有点隐蔽，因为a不是对象，instanceof时，它被转成包装对象，那么①处当然是true，不过a始终只是个字符串，所以②处自然是false
// 因为instanceof不能直接判断俩个对象之间的关系，所以可以使用下面这个方法
// 不过得看理解角度，在类这个角度而言，很是荒谬，毕竟o1很明显没有继承F，不过判断对象关联还是有可取的
function isRelatedTo(o1, o2) {
    function F() {}
    F.prototype = o2;
    return o1 instanceof F;
}
var a = {};
var b = Object.create(a)
isRelatedTo(b, a) // true
```
> 如果右值是用.bind()生成的硬绑定函数，那么是没有.prototype属性的，这时候不会报错，会使用目标函数的.prototype属性(其实实际上相当于直接调用目标函数，instanceof也就相当于直接使用目标函数)
+ ``Bar.prototype.isPrototypeOf(bar)``：这个好处很明显，无需考虑是不是函数对象。```a.isPrototypeOf(b) // a是否出现在b的原型链之上```，就是上文的``isRelatedTo``方法
PS：\_\_proto\_\_这个属性不是兼容所有的浏览器（在ES6之前不是标准）。它存在``Object.prototype``上。大致如下
```javascript
Object.defineProperty(Object.prototype, '__protp__', {
    get() {
        return Object.getPrototypeOf(this)
    },
    set(o) {
        Object.setPrototypeOf(this, o)
    }
})
```
# 对象关联
嗯，之前说的```__proto__```准确的描述应该是[[Prototype]]机制，毕竟这个属性在ES6之前不是标准，不是所有的浏览器都提供
1. 创建关联
```Object.create```。
值得注意的是，``Object.create(null)``会创建一个空[[Prototype]]链接对象。因为没有原型链，所以```instanceof```无法判断。这个对象也被叫做**字典**，不受原型链干扰，适合存储数据
2. 关联关系是备用
关联关系可以是处理对象缺失属性或者非法的一种备用选项。不过这可能并不是个好事。比如你本来没有设计这个方法，结果居然可以运行，那么不就很神奇了。就像如下：
```javascript
var anotherObject = {
    output: function () {
        console.log('output')
    }
};
var myObject = Object.create(anotherObject)
myObject.output() // output
```
其实可以修改下（委托设计模式），就是在实际存在的方法里面委托到别的方法处理，**内部委托比直接委托可以让API接口更清晰**
```javascript
var anotherObject = {
    output: function () {
        console.log('output')
    }
};
var myObject = Object.create(anotherObject)
myObject.doOutput = function() {
	this.output()
}
myObject.doOutput() // output
```
# 常用polyfill
1. new
```javascript
function New() {
    var args = Array.prototype.slice.call(arguments)
    var F = args.shift()
    var o = {}
    o.__proto__ = F.prototype

    var res = F.apply(o, args)
    if ((typeof res === 'object' || typeof res === 'function') && res !== null) {
        return res
    }
    return o
}
function Car(year) {
    this.year = year;
    // return function () { }
}
var car1 = New(Car, 2019)
car1
// 使用Object.create
function New() {
    var args = Array.prototype.slice.call(arguments)
    var F = args.shift()
    var o = Object.create(F.prototype)
    var res = F.apply(o, args)
    if ((typeof res === 'object' || typeof res === 'function') && res !== null) {
        return res
    }
    return o
}
function Car(year) {
    this.year = year;
    // return function () { }
}
var car1 = New(Car, 2019)
car1
```
2. Object.create
```javascript
if (!Object.create) {
    Object.create = function (o) {
        function F() { }
        F.prototype = o
        return new F()
    }
}
```
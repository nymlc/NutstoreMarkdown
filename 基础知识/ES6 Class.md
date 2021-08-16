# super（1）
首先说下super。在ES6之前，并没有方法（method）这个概念，都是对象的一个属性值。不过到了ES6就有了。它拥有[[HomeObject]]这个属性，是创建之时静态绑定的。super就是指向[[HomeObject]].[[Prototype]]
```javascript
// 很明显print的[[HomeObject]]是obj，是不会改变的
const obj = {
    name: 'obj',
    print() {
        console.log(this.name, super.name)
    }
}
// 这时候obj的prototype对象还没有name属性
obj.print() // obj undefined

Object.setPrototypeOf(obj, {name: 'super'})
// 这时候obj的prototype对象name属性为super
obj.print() // obj super

const print = obj.print
// 这时候this指向为window
print() // '' super
```
# super（2）
其实远远没这么简单，不过其实基本也够用。我们比较打破砂锅的看看super
1. 为什么需要[[HomeObject]]，this不行吗？
其实我们可能觉得this也够用，毕竟不就是查询到'父类'对象，然后调用对应的方法么。其实没这么简单
```javascript
// super版
var A = {
    name: 'A',
    print() {
        // 这里this相当于C，就像是从C处层层代入一样
        console.log(`${this.name} print.`)
    }
}
var B = {
    name: 'B',
    print() {
        // 这里super相当于B.__proto__（A），即A.print()
        super.print()
    }
}
Object.setPrototypeOf(B, A)
var C = {
    name: 'C',
    print() {
        // 这里super相当于C.__proto__（B），即B.print()
        super.print()
    }
}
Object.setPrototypeOf(C, B)
C.print() // C print.

// this版 
var A = {
    name: 'A',
    print() {
        console.log(`${this.name} print.`)
    }
}
var B = {
    name: 'B',
    print() {
        // 这里this指代C，相当于B.print.call(C)
        this.__proto__.print.call(this)
    }
}
Object.setPrototypeOf(B, A)
var C = {
    name: 'C',
    print() {
        // 这里this指代C，相当于B.print.call(C)
        this.__proto__.print.call(this)
    }
}
Object.setPrototypeOf(C, B)
// 相当于一直调用B.print
C.print() // VM1807:9 Uncaught RangeError: Maximum call stack size exceeded
```
可以看出，this版动态绑定，它会变化，导致无限调用。super版[[HomeObject]]创建之时静态绑定，不可修改。可以解决这个this无法解决的问题
> 值得注意的是，通过super调用的上层函数，this是被代入的，就像call传参代入一样

2. Class里的super
```javascript
class A {
    name = 'A'
    print() {
        console.log(`${this.name} print！`)
    }
}
class B extends A {
    name = 'B'
    print() {
        // 这里的super指代的是B.prototype.__proto__即A.prototype，因为这个print是属于B.prototype
        super.print()
    }
}
(new B).print()
```
3. babel转化上面的this版。这个可以自行去看看。它是显示根据宿主对象溯源的。
# 转换差异
```javascript
class Foo extends null {}
```
这个也是合法的。不过并不能new一个对象出来，因为null没有constructor
不过若是babel转换的是可以new的。这里得说下关键的extends做了什么。它主要是做了以下俩点：
```javascript
Object.setPrototypeOf(subClass, superClass)
Object.setPrototypeOf(subClass.prototype, superClass.prototype)
```
不过，extends null时，只做了第二点，所以看源码并不报错
# Class babel的转换对比
```javascript
// ES6
class Foo {
    static age = 20
    sex = '1'
    constructor(name) {
        this.name = name
    }
	printAge() {
    	console.log(this.age)
    }
	static printName() {
    	console.log(this.name)
    }
}

class Bar extends Foo {
    constructor(name, label) {
        super(name)
        this.label = label
    }
    printAge() {
        super.printAge()
        console.log('Bar printAge')
    }
	static printName() {
        super.printName()
        console.log('Bar printName')
    }
}
// ES5
"use strict";
// 为了处理Symbol不可用情况下的typeof报错问题
function _typeof(obj) {
    // 若是支持的话，那么Symbol.iterator typeof检测是symbol
    if (typeof Symbol === "function" && typeof Symbol.iterator === "symbol") {
        _typeof = function _typeof(obj) {
            return typeof obj;
        };
    } else {
        _typeof = function _typeof(obj) {
            // 不支持的话，那么就是引用了babel-polyfill之类的兼容库
            // 这种兼容库不能做到的是typeof检测下不可能有symbol这个基础数据类型
            // typeof Symbol === "function" && obj.constructor === Symbol && obj !== Symbol.prototype（Symbol方法存在、obj由Symbol构造、obj不是Symbol.prototype），满足三点，那么必然是symbol
            return obj && typeof Symbol === "function" && obj.constructor === Symbol && obj !== Symbol.prototype ? "symbol" : typeof obj;
        };
    }
    return _typeof(obj);
}
// 这个其实是用于处理诸如父类构造函数有返回（call）的情况
// this绑定的new绑定，构造函数有返回对象的话
function _possibleConstructorReturn(self, call) {
    if (call && (_typeof(call) === "object" || typeof call === "function")) {
        return call;
    }
    return _assertThisInitialized(self);
}
// 这里应该是多判了，因为这个self必定是子类的实例，是不可能为undefined的
function _assertThisInitialized(self) {
    if (self === void 0) {
        throw new ReferenceError("this hasn't been initialised - super() hasn't been called");
    }
    return self;
}
// 获取目标对象的某值
function _get(target, property, receiver) {
    // 使用Reflect就是为了方便设置get里的this（receiver）
    if (typeof Reflect !== "undefined" && Reflect.get) {
        _get = Reflect.get;
    } else {
        _get = function _get(target, property, receiver) {
            var base = _superPropBase(target, property);
            if (!base) return;
            var desc = Object.getOwnPropertyDescriptor(base, property);
            if (desc.get) {
                // 如果target对象中指定了getter，receiver则为getter调用时的this值
                return desc.get.call(receiver);
            }
            return desc.value;
        };
    }
    return _get(target, property, receiver || target);
}

/**
 *获取属性的宿主
 *
 * @param {*} object 目标对象
 * @param {*} property 目标属性
 * @returns
 */
function _superPropBase(object, property) {
    while (!Object.prototype.hasOwnProperty.call(object, property)) {
        object = _getPrototypeOf(object);
        if (object === null) break;
    }
    return object;
}

/**
 *获取对象的.prototype属性
 *
 * @param {*} o
 * @returns
 */
function _getPrototypeOf(o) {
    _getPrototypeOf = Object.setPrototypeOf ? Object.getPrototypeOf : function _getPrototypeOf(o) {
        return o.__proto__ || Object.getPrototypeOf(o);
    };
    return _getPrototypeOf(o);
}

/**
 *继承的核心代码，其实就是对象关联
subClass.__proto__ === superClass
subClass.prototype.__proto__ === superClass.prototype
 *
 * @param {*} subClass 子类
 * @param {*} superClass 父类
 */
function _inherits(subClass, superClass) {
    // 父类必须是function或者null
    if (typeof superClass !== "function" && superClass !== null) {
        throw new TypeError("Super expression must either be null or a function");
    }
    // 链接子类和父类的.prototype
    // Object.create第二个参数为设置返回值的本身属性及其描述符，这里是修复子类.prototype的.constructor属性
    subClass.prototype = Object.create(superClass && superClass.prototype, {
        constructor: {
            value: subClass,
            writable: true,
            configurable: true
        }
    });
    // 链接子类和父类对象
    // 用于子类可获取父类静态方法和变量
    if (superClass) _setPrototypeOf(subClass, superClass);
}

/**
 *setPrototypeOf polyfill
 *
 * @param {*} o 子类
 * @param {*} p 父类
 * @returns
 */
function _setPrototypeOf(o, p) {
    _setPrototypeOf = Object.setPrototypeOf || function _setPrototypeOf(o, p) {
        o.__proto__ = p;
        return o;
    };
    return _setPrototypeOf(o, p);
}

// Symbol.hasInstance这个用于判断对象是否是某个构造器实例
// 它存在于Function.prototype上，所以只要是函数对象就都可以访问到它，所以想要给一个函数对象重新定义这个方法，那么得用Object.defineProperty
/*
function Foo() {

}
Object.defineProperty(Foo, Symbol.hasInstance, {
    value: function value(instance) {
        return true
    },
    enumerable: true,
    configurable: true
})

'' instanceof Foo  // true
*/
function _instanceof(left, right) {
    if (right != null && typeof Symbol !== "undefined" && right[Symbol.hasInstance]) {
        return !!right[Symbol.hasInstance](left);
    } else {
        return left instanceof right;
    }
}
// 做了个判断，确保它被构造调用
function _classCallCheck(instance, Constructor) {
    if (!_instanceof(instance, Constructor)) {
        throw new TypeError("Cannot call a class as a function");
    }
}

/**
 *给对象赋值
 *
 * @param {*} target 赋值的对象
 * @param {Array} props 要赋的值的描述对象
 */
function _defineProperties(target, props) {
    for (var i = 0; i < props.length; i++) {
        var descriptor = props[i];
        descriptor.enumerable = descriptor.enumerable || false;
        descriptor.configurable = true;
        if ("value" in descriptor) descriptor.writable = true;
        Object.defineProperty(target, descriptor.key, descriptor);
    }
}

/**
 *给”类“赋方法
 *
 * @param {*} Constructor ”类“
 * @param {Array} protoProps 普通方法（挂在.prototype上，”实例“可访问）
 * @param {Array} staticProps 静态方法（挂在”类“上，"类"可访问）
 * @returns
 */
function _createClass(Constructor, protoProps, staticProps) {
    if (protoProps) _defineProperties(Constructor.prototype, protoProps);
    if (staticProps) _defineProperties(Constructor, staticProps);
    return Constructor;
}
// 给对象赋值，若是已存在，则使用Object.defineProperty
function _defineProperty(obj, key, value) {
    if (key in obj) {
        Object.defineProperty(obj, key, {
            value: value,
            enumerable: true,
            configurable: true,
            writable: true
        });
    } else {
        obj[key] = value;
    }
    return obj;
}
// 可见，ES6的class其实就是个function的语法糖
var Foo =
    /*#__PURE__*/
    function () {
        function Foo(name) {
            // 做了个判断，确保它被构造调用
            _classCallCheck(this, Foo);
            
            _defineProperty(this, "sex", '1');
            // 这里其实已经是constructor范畴了
            this.name = name;
        }
        // 给”类“赋方法
        _createClass(Foo, [{
            key: "printAge",
            value: function printAge() {
                console.log(this.age);
            }
        }], [{
            key: "printName",
            value: function printName() {
                console.log(this.name);
            }
        }]);

        return Foo;
    }();
// 赋值静态属性
_defineProperty(Foo, "age", 20);
// 这里使用立即执行函数创造一个沙箱，可以避免内部变量污染外部
var Bar =
    /*#__PURE__*/
    function (_Foo) {
        _inherits(Bar, _Foo);

        function Bar(name, label) {
            var _this;
            
            // 做了个判断，确保它被构造调用
            _classCallCheck(this, Bar);
            // 调用父类构造函数，获取其返回值，若是有返回值，那么就是新的this，否则还是现有的this
            // 因为若是有返回值，那么父类构造器返回的就是该返回值，那么子类也得赋值在此对象上，不然俩边不一致
            // 若是extends null，_getPrototypeOf(Bar)就是Function.prototype
            _this = _possibleConstructorReturn(this, _getPrototypeOf(Bar).call(this, name));
            _this.label = label;
            return _this;
        }

        // 给”类“赋方法
        _createClass(Bar, [{
            key: "printAge",
            value: function printAge() {
                // 因为这个不是静态方法，所以在Bar.prototype的父级上取
                _get(_getPrototypeOf(Bar.prototype), "printAge", this).call(this);

                console.log('Bar printAge');
            }
        }], [{
            key: "printName",
            value: function printName() {
                // 因为这个是静态方法，所以在Bar的父级上取
                _get(_getPrototypeOf(Bar), "printName", this).call(this);

                console.log('Bar printName');
            }
        }]);

        return Bar;
    }(Foo);
```
# 面向委托的设计
其实就是jQuery那一套
1. 类理论
类的套路就是定义一个基类，里面包含了所有通用的行为，然后在子类添加一些自己特殊的行为
类设计鼓励重写父类方法
2. 委托理论
委托的套路就是定义一个基本对象，它包含了所有的通用行为，然后在具体的对象（子类）里存储对应的特殊行为，关联俩对象，这样子可以委托通用行为
```javacript
var Task = {
    setName(name) {
        this.name = name
    },
    outputName() {
        console.log(this.name)
    }
}
var MajorTask = Object.create(Task)
MajorTask.prepareTask = function (name, label) {
    this.setName(name)
    this.label = label
}

MajorTask.outputDetails = function () {
    this.outputName()
    console.log(this.label)
}

var task1 = Object.create(MajorTask)
task1.prepareTask('task1', 'first')
task1.outputDetails()
```
这种编码风格叫做**对象关联**相对面向对象而言，相对类设计的重写而言，我们不推荐使用相同的命名，因为需要使用显示伪多态这种难看的语法
它和类设计父类到子类这种垂直的关系不同的是，它是任意方向的委托。**最好不要直接委托，内部委托最好**
+ 互相委托（禁止）：如果引用的是俩者都没有的属性不就无限递归循环了。
+ 调试：在Chrome控制台上，若是一个对象是”构造函数调用“生成的，那么打印的结果肯定是诸如``Bar {name: "bar", label: "child"}``。它其实想说的是这个对象是由名为Bar（.constructor.name）的函数构造。
其实Chrome并不是直接输出对象的``.constructor.name``。就比如一个经典原型继承例子（之前的Foo、Bar这个例子，bar.constructor.name是Foo，打出来的显示还是Bar {...}）。不过也是有意外（这个其实是Chrome的bug）
```javascript
var Foo = {
	constructor: function Gotcha(){}
}
var a1 = Object.create( Foo )
a1 // Gotcha {}
```
这个功能其实是Chrome的一种扩展功能，不包含在JS规范之中。如果不是“构造函数调用”生成的对象那么是无法跟踪“构造函数名称”的。这个在类设计里有用，需要考虑对象由谁构造，委托设计不需要，因为根本不用关注是谁构造了对象
3. 比较思维模型
原型继承
```javascript
function Foo(me) {
    this.me = me
}
Foo.prototype.identify = function () {
    return `I am ${this.me}`
}

function Bar(me) {
    Foo.call(this, me)
}

Bar.prototype = Object.create(Foo.prototype)

Bar.prototype.speak = function() {
    console.log(`hello, ${this.identify()}.`)
}
var b1 = new Bar('b1')
var b2 = new Bar('b2')
b1.speak()
b2.speak()
```
![原型继承](https://upload-images.jianshu.io/upload_images/4874009-480e858f35031e81.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
委托设计
```javascript
var Foo = {
    init(me) {
        this.me = me
    },
    identify() {
        return `I am ${this.me}`
    }
}

var Bar = Object.create(Foo)

Bar.speak = function () {
    console.log(`hello, ${this.identify()}.`)
}

var b1 = Object.create(Bar)
b1.init('b1')
b1.speak()
var b2 = Object.create(Bar)
b2.init('b2')
b2.speak()
```
![委托设计](https://upload-images.jianshu.io/upload_images/4874009-a1ce1486c7c8584b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 类与对象
1. 控件“类”
+ ES6之前的JS类
```javascript
function Widget(width, height) {
    this.width = width
    this.height = height
    this.$ele = null
}

Widget.prototype.render = function ($where) {
    const {
        width,
        height
    } = this
    if (this.$ele) {
        this.$ele.css({
            height,
            width
        }).appendTo($where)
    }
}

function Button(width, height, label) {
    // 显示伪多态
    Widget.call(this, width, height)
    this.label = label
    this.$ele = $('<button></button>').text(label)
}

Button.prototype.render = function ($where) {
    // 显示伪多态
    Widget.prototype.render.call(this, $where)
    this.$ele.click(this.onClick.bind(this))
}
Button.prototype.onClick = function () {
    console.log(`Button：${this.label}被点击！`)
}

$(document).ready(function () {
    const $body = $(document.body)
    const button1 = new Button(100, 80, '保存')
    const button2 = new Button(100, 80, '新建')
    button1.render($body)
    button2.render($body)
})
```
很明显的缺点就是显示伪多态
+ ES6的class
```javascript
class Widget {
    constructor(width, height) {
        this.width = width
        this.height = height
        this.$ele = null
    }
    render($where) {
        const {
            width,
            height
        } = this
        if (this.$ele) {
            this.$ele.css({
                height,
                width
            }).appendTo($where)
        }
    }
}
class Button extends Widget {
    constructor(width, height, label) {
        super(width, height)
        this.label = label
        this.$ele = $('<button></button>').text(label)
    }
    render($where) {
        // 值得注意的是，这里不用super.render.call(this)
        super.render($where)
        this.$ele.click(this.onClick.bind(this))
    }
    onClick() {
        console.log(`Button：${this.label}被点击！`)
    }
}

$(document).ready(function () {
    const $body = $(document.body)
    const button1 = new Button(100, 80, '保存')
    const button2 = new Button(100, 80, '新建')
    button1.render($body)
    button2.render($body)
})
```
它仅仅是ES6的语法糖，不过是让你编码体验好一点
2. 委托控件对象
```javascript
var Widget = {
    init(height, width) {
        this.width = width
        this.height = height
        this.$ele = null
    },
    render($where) {
        const {
            width,
            height
        } = this
        if (this.$ele) {
            this.$ele.css({
                height,
                width
            }).appendTo($where)
        }
    }
}
var Button = Object.create(Widget)
Object.assign(Button, {
    setup(width, height, label) {
        this.init(width, height)
        this.label = label
        this.$ele = $('<button></button>').text(label)
    },
    build($where) {
        this.render($where)
        this.$ele.click(this.onClick.bind(this))
    },
    onClick() {
        console.log(`Button：${this.label}被点击！`)
    }
})

$(document).ready(function() {
    const $body = $(document.body)
    const button1 = Object.create(Button)
    button1.setup(100, 80, '保存')
    button1.build($body)
    const button2 = Object.create(Button)
    button2.setup(100, 80, '新建')
    button2.build($body)
})
```
可以看到的是，这个就很符合JS的万物皆对象。这里没有类。每个对象有自己的属性、方法。若是没有的话可以委托给别的对象
+ 我们在俩个对象里没有定义一样的方法名，而是取了更具有描述性的方法名（render、build）
+ 我们通过this以简单的相对委托（this.init）来避免显示伪多态
+ 之前的``new Button()``现在分作俩步（构造、初始化）。其实它们俩不需要合并为一个步骤
> 比如有个实例池，这时候很明显，你需要用实例的时候初始化更好。
# 更简洁的设计
其实这么多以来，俩者设计孰优孰劣自有定论，自认为委托模式的确更简单点。
俩者差异还有一点就是“类”设计需要个基类，委托设计就不用了，可以任意委托
# 更好的语法
想说的是ES6的简洁方法声明。不过它有个小缺点，那就是去了语法糖之后基本上相当于匿名函数。就像之前（作用域和闭包）说的匿名函数三大缺点
1. 执行栈难追踪
2. 自我调用（递归、事件解绑）
3. 代码缺失自注释
简洁方法没有1、3俩点缺陷。其中第一点主要是简洁方法有点特殊，会给对应的函数对象设置一个内部的name属性，这样子理论是可以用于追踪的（实际上追踪的实现并不统一，所以也看情况不见得能用）
```javascript
const Foo = {
    doSomething() {
        
    }
}
```
# 自省
自省在Java里很常见，其实就是**检查下实例的类型**，目的就是通过创建方式来判断对象的结构和功能
+ instanceof
```javascript
function Foo() {
    // do something
}
Foo.prototype.something = function () {
    // do something
}
var a = new Foo()
// 自省
if (a instanceof Foo) {
    a.something()
}
```
之前所述，a的原型链上有Foo.prototype，也就是相当于告诉我们a（对象）是Foo（类）的实例，那么自然可以使用Foo的something方法
不过``instanceof``会产生困惑，它好像判断的是a和Foo之间的关系，不过显然并不是。它不能直接判断俩对象是否关联。就像你想检查a和Foo的关系，那么就必须使用另一个***a委托的对象***（Foo.prototype）才可以
```javascript
function Foo() {}
Foo.prototype.something = function () {}
function Bar() {}
// 这点很重要，和对象关联风格的重要区别
Bar.prototype = Object.create(Foo.prototype)
var b1 = new Bar()

// 让Foo和Bar互相关联（Foo和Bar之间的继承（委托）关系）
Bar.prototype instanceof Foo // true 
Object.getPrototypeOf(Bar.prototype) === Foo.prototype // true
Foo.prototype.isPrototypeOf(Bar.prototype) // true

// 让b1关联到Foo和Bar
b1 instanceof Foo // true
b1 instanceof Bar // true
Object.getPrototypeOf(b1) === Bar.prototype // true 
Foo.prototype.isPrototypeOf(b1) // true 
Bar.prototype.isPrototypeOf(b1) // true
```
这里不好的点其实主要是你判断Bar和Foo的关系，你想的是``Bar instanceof Foo``，但是显然并不可以
+ 鸭子类型
如果看起来像鸭子，叫起来像鸭子，那就一定是鸭子
```javascript
if (a.something) {
    a.something()
}
```
我们没有检查a和委托something函数的对象（Foo.prototype）之间的关系。而是假设a有something（无论是存在a自身还是通过委托）。**其实这个假设风险并不高**
ES6的Promise就是典型的“鸭子类型”。比如我们要判断一个对象的引用是否是Promise，我们就检查对象是否有then方法，有的话ES6的Promise就会认为这个对象是thenable（可持续）的，就会期望它具有Promise的所有标准行为
很明显，这个危害也有。就像一个普通对象具有then方法，**使用的时候要保证可控**
+ isPrototypeOf
```javascript
var Foo = {
    a: 123
}
// 这点很重要，和类风格的重要区别
var Bar = Object.create(Foo)
// 就  =   Foo是Bar的原型吗？
Foo.isPrototypeOf(Bar) // true
Object.getPrototypeOf(Bar) === Foo; // true
```

# 一些术语
1. 抽象
抽象是指为了某种目的，对一个概念或一种现象包含的信息进行过滤，移除不相关的信息，只保留与某种最终目的相关的信息。从另外一个角度看，抽象就是简化事物，抓住事物本质的过程
抽象是分层次的：
+ 我昨天的袜子
+ 昨天的袜子
+ 一双袜子
+ 纺织物
在不同的层次上抽象就是过滤不同的信息，我们需要确保最终留下来的信息是所有抽象层需要的信息
2. 关注点分离
一个任务很复杂，要关注的点太多，人的精力又有限，那么把一个任务细分很重要。
实现关注点分离有俩种方法：标准化、抽象包装
+ 标准化：就像ECMAScript，我们只要关注标准的本身以及我们想要做什么就行了，大家都准守这个标准，那么我们的程序和别人的程序就能配合起来
+ 抽象与包装：就像把很多UI库就是在不同的层次（按钮啥的）上做了抽象和包装，我们不用关注内在实现
**其实就是高内聚，低耦合**
3. 池
什么是**“池”**呢？其实想想线程池就好了。你的应用经常需要线程实例，那么你本来做法是起个线程，完事了销毁。但是起线程和销毁线程是耗资源的。那么你事先起几个放池里，用的时候拿出来，不用了放进去。
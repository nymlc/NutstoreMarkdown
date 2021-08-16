首先可以看下ES5对ES6 [Class的实现（Babel)](https://www.jianshu.com/p/19b6e0a0a446)，这里也包含了一些ES6 class的一些原理
第二部分主要就说了一点：“类设计”是可选的设计模式，在JS中实现类并不好
这种不好不仅仅来自于语法，繁乱的``.prototype``引用、显示伪多态、不可靠的``.constructor``。而且类的本质是复制，这里并没有，有的仅仅是委托
# class
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
```
1. 没有繁乱的``.prototype``
2. 可以用super来实现相对多态
3. 声明Button时就直接继承了Widget，不用修改``.prototype``
4. 可以通过extends很自然的扩展对象子类型，甚至是Array之类的内置对象
5. 值得注意的是可以声明实例属性，和方法平级，也可以和上例一样挂在this上
# class陷阱
首先最大的误解就是不能一味ES6引入了新机制，它仅仅是之前我们写的``.prototype``“类设计”的语法糖，本质上它还是个对象。所以声明的类是可以被改的，而且父类改了，子类也被影响，它本质上还是没有实现**复制**
1. class语法有可能出现意外屏蔽
```javascript
class C {
    constructor(id) {
        // id 属性屏蔽了 id() 方法
        this.id = id
    }
    id() {
        console.log('Id: ' + id)
    }
}
var c1 = new C('c1')
c1.id() // // TypeError -- c1.id 现在是字符串 "c1"
```
2. super的绑定。可能我们会认为它和this一样是动态绑定。但是其实动态绑定很耗性能，而super在在声明的时候“静态”绑定。它指向[[HomeObject]].[[Prototype]]，[[HomeObject]]指向方法被调用的对象，静态绑定，不能修改

>class不能extends一个没有有效的prototype属性的函数，比如被bind过得函数
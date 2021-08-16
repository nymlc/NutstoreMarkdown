主要讲三个array、string、number
# 数组
+ delete可以删除数组某单元，不过删除之后length是不会变的
+ 创建稀疏数组（含有空白或空缺的数组）要注意未设置的单元输出值显示为undefined，不过和显式赋值为undefined还是有区别的
+ 数组索引若是可以被强制类型转换为十进制数字的话是可以被当做数字索引使用
#### 类数组
类数组是一组通过数字索引取值的对象且具有length属性，很多时候需要将其转换为真正的数组，常用的有：
+ ```Array.prototype.slice.call(arguments)```
+ ```Array.from(arguments)```
> 其实类数组就是具有length属性的就算是了，就像```function(){}```

>为什么需要类数组呢？它具有数组一些性质，还可以对它的原型进行扩展而不会影响到Array，相对数组可扩展性好
# 字符串
**字符串是个类数组**，不过这个**类数组**是不可变的。
```javascript
var a = '123'
// 这里在低版本IE是不合法的，应该是a.charAt(1)
a[1] = 4
a // 123
```
字符串是不可变的，就是说字符串的成员函数不会改变字符串的原始值，都是返回一个新的字符串，不像数组。**所以reverse是对字符串无用的**
> 可以借用数组的一些方法来处理字符串，就像```Array.prototype.join.call(a, '-')```
# 数字
# 特殊数值
1. 不是值的值
null和undefined既是类型也是值
+ null指的是空值
+ undefined指的是未被赋值
+ null不能被当做变量来赋值和引用，undefined可以
2. undefined
在非严格模式下，全局的undefined可以被重新赋值（也不是每个浏览器都能修改成功，不过不报错），严格模式下是会抛出异常
![image.png](https://upload-images.jianshu.io/upload_images/4874009-c275dbbe5e547978.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在严格和非严格模式下，undefined可以作为变量（很糟糕的设定，没啥用）
#### viod运算符
undefined是一个内置的标识符，值是undefined，它可以用void运算符计算得来
**void __**没有返回值，所以返回的是undefined，它对操作值没有副作用，只是让表达式不返回值而已
一般而言我们都是```void 0```，其实```void 1、void true```都是一样的
个人一般都不用，不过看压缩过的代码多包含这个，也能避免undefined被篡改
3. 特殊的数字
#### 不是数字的数字
如果数学运算的操作符不是数字类型（或者不能转换成数字），那么就会返回NaN（not a number），其实理解为无效的数值更准一些。**typeof返回值是number**
+ NaN是一个很特殊的值，是唯一一个非自反值。即```NaN !== NaN // true```
+ isNaN可以判断一个值是否是NaN，不过有个坑，就是**它会对检查的这个值进行强转，然后看这个值是否是NaN**，这样子的话一些字符串也就是会返回true了，建议使用```Number.isNaN```，或者判断一下这个值是不是number，或者根据它的非自反特性（```return n !== n```）
#### 无穷数
#### 零值
-0是个常量，它和+0是相等的。**加法和减法是不能得到-0**
将-0转成字符串是'0'，不过将'-0'转成数字却是-0
```javascript
var a = -0
a.toString() // '0'
a + '' // '0'
String(a) // '0'
JSON.stringify(-0)  // '0'

var b = '-0'
+b // -0
Number(b) // -0
JSON.parse(b) // -0
parseInt(b) // -0
parseFloat(b) // -0
```
判断-0
```
function isNegZero(n) {
    n = +n
    return n === 0 && 1 / n === -Infinity
}
isNegZero(0) // false
isNegZero(-0) // true
```
为什么需要-0呢？比如有时候需要判定速度的变化（有加减这个方向），+-就是这个值的方向信息，若是0没有+-的话，那么方向信息就丢失了。**其实就是判定在这个0值的时候变化的方向**
4. 特殊等式
NaN和-0在上文可以看出在相等比较时有点特殊，其实除了以上说的还可以用ES6的```Object.is```
```javascript
const a = NaN
const b = -0

Object.is(a, NaN) // true
Object.is(b, -0) // true
```
polyfill
```javascript
Object.is = function(a, b) {
    // -0
    if (a === b && a === 0) {
        return 1 / a === 1 / b
    }
    // NaN
    if (a !== a) {
        return b !== b
    }
    return a === b
}
```
# 值和引用
这俩有疑惑多出于传参。传参分为值复制和引用复制，我们无法自行决定，它取决于值的类型
+ 引用传参就是将复合值的**引用副本**传递进函数，函数内部对这个引用的值进行操作的话，那么函数外部被传进去的的引用对应的值自然也会发生改变。不过若是函数内部的引用副本指向改变了，那么原引用对应的值自然是无变化的
```javascript
function foo(arr) {
    arr.push(4)
    arr // [1, 2, 3, 4]

    arr = [4, 5, 6]
    arr.push(7)
    arr // [4, 5, 6, 7]
}
var arr = [1, 2, 3]
foo(arr)
arr // [1, 2, 3, 4]
```
+ 若是传参是复合值，想通过值传参，那么可以复制这个复合值```arr.slice()```
+ 若是想将非复合值传递进函数内修改（达到内部修改，外部同步变化），那么就得将这个基本值封装到一个复合值，然后引用复制的方式传递就行了
+ 若是将一个基本类型值封装成它的封装对象，然后引用传递可能会和想得有点不一样
```javascript
function foo(x) {
    // 这里的x已经是偷偷地从引用对象变成了基本值，外部的b引用指向不变，还是1
    x++
    x // 2
}
var a = 1
var b = new Number(a)
foo(b)
b // 1
```
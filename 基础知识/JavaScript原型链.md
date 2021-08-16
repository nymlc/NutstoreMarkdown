# 对象
按typeof而言，对象有**function、object**。我们称function为函数对象，object为普通对象。
其实俩者区别在于函数对象为new Function()创建
>Object Function都是通过 new Function()创建的
# 构造属性/constructor
函数对象的实例均有构造函数属性(constructor)
```javascript
function person() {
	
}
var p = new person();
console.log(p.constructor === person); // true
```
> 其实除了null、undefined都有constructor，值得注意的是诸如```var a = 3```，a是包装对象，是Number这个函数对象的一个实例，constructor就是指向实例的构造函数
# 原型对象/prototype
只要是对象定义了都会都会包含一些预定的属性，即使```var o = {}```，o也会有\_\_proto\_\_之类的属性。其中函数对象拥有prototype属性，该属性指向此函数对象的原型对象
其实原型对象就是一个普通对象，typeof下均是object除了Function的原型对象。原型对象是其对应的对象的实例（instanceof是不过的，伪实例）
```javascript
Person.prototype <==> new Person();
// 这是因为原型对象由对应的构造函数构造
Person.prototype.constructor === Person; // true
```
值得注意的是，原型对象默认情况下均有constructor属性（它是一个指针），理所应当，因为其为函数对象实例，所以其自然指向该函数对象
>1. 函数对象才有prototype属性，它指向原型对象
>2. 原型对象是对应的函数对象的实例(所以它也有constructor属性)
>3. Function的原型对象typeof下是function，最特殊的一个，原因就是其原型对象为Function的一个实例，那么自然就是函数对象，虽然它是函数对象，不过特别的是它并没有prototype属性
# 原型链\_\_proto\_\_
只要是对象均有\_\_proto\_\_这个属性。它指向它的构造函数的原型对象
```javascript
person1.__proto__ === Person.prototype;
Person.prototype.constructor == Person;
```
![红宝书第六章](https://upload-images.jianshu.io/upload_images/4874009-71977c3b3b5dae51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>\_\_proto\_\_只是浏览器支持，但是不是ES5标准，不过加入了ES6
# 注意点
1. ```Object.__proto__ === Function.prototype // true```
Object是函数对象，通过constructor可知
2. ```Function.__proto__ === Function.prototype // true```
Function也是函数对象，也就是Function也是new Function()创建，
3. ```Function.prototype.__proto__ === Object.prototype //true```
这点是很特殊的，按理Function.prototype是函数对象，它的\_\_proto__应该指向Function.prototype，不过万物从无到有，函数对象也是对象，指向Object.prototype，然后```Object.prototype.__proto__ === null```，使得原型链指向null

![常见指向图](https://upload-images.jianshu.io/upload_images/4874009-ea1c6a0deaea7ae2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

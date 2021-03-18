---
title: JS继承
date: 2021-03-16 23:16:35
tags:
---

构造函数(用来创建对象的函数)与普通函数本质上都是函数.它们的区别在于:

- 命名上构造函数的首字母建议大写,而普通函数首字母建议小写
- 调用方式不同,普通函数直接调用,构造函数需要使用`new`关键字调用
- 构造函数内部`this`指向创建的实例,普通函数的`this`指向最后一个调用它的对象
- 构造函数中没有显示的`return`表达式,一般是隐士的返回`this`也就是新创建的对象,如果想要显示的返回值,则需要返回一个对象,否则依然是返回实例. 普通函数没有`return`默认返回`undefined`

使用构造函数时:
```
function Animal(){} // 声明一个函数(构造函数)
vat animial = new Animal() // 使用这个函数创建对象
// 等价于
function Animal(){} // 声明一个函数
var animal = {} // 声明定义一个对象
animal.__proto__ = Animal.prototype // 将对象的隐士原型指向函数的原型对象
var res = Animal.call(animal); // this指向animal
return typeOf res === 'object' ? res : animal // 如果res非对象或者为空,则输出animal,否则输出res
```

### prototype

创建函数的时候,函数都会有一个`prototype`属性,指向函数的原型对象(本质也是对象),而这个原型对象上有一个`constructor`属性指向这个函数(通常是,可以被变更).
当函数作为构造函数时,所有实例对象需要共享的方法和属性都放在`prototype`,不需要共享的属性和方法放入构造函数中.实例对象一旦创建,将自动引用`prototype`中的属性和方法.由于所有的实例对象共享同一个`prototype`对象,那么从外界看来,`prototype`对象就好像是实例对象的原型,而实例对象好像是“继承”了`prototype`对象一样

### __proto__

`__proto__`已从Web标准中删除,但是一些浏览器目前仍然支持它.它是存在于每个对象中的属性(浏览器提供的).

> 通过现代浏览器的操作属性的便利性,可以改变一个对象的[[prototype]]属性,这种行为在每一个Javascript引擎和浏览器中都是一个非常慢且影响性能的操作,使用这种方式来改变和继承属性对性能的影响是非常严重的,并且性能消耗的时间也不是简单的花费在 obj.__proto__ = ... 语句上,它还会影响到所有继承来自该[[prototype]]的对象.
> __proto__的设置器(setter)允许对象的[[prototype]]被变更,前提是这个对象必须通过 Object.isExtensible() 判断为可扩展的,否则抛出一个 TypeError 的异常.要变更的值必须是一个对象或者null,提供其他值将不起作用.
> __proto__的读取器(getter)暴露了一个对象内部的[[prototype]].对于使用对象字面量的对象,这个值就是Object.prototype.对于使用数组字面量创建的对象,这个值就是 Array.prototype.对于函数来说,这个值就是 Function.prototype.当使用new操作符创建的对象时,无论构造函数是JS提供的内置构造函数(Array、Boolean、Date、Number、Object、String等)还是使用JS自定义的构造函数,那么这个值都是这个构造函数的prototype.

`__proto__`和`prototype`的区别在于`__proto__`存在于对象中而`prototype`只存在于函数中.函数是对象,但对象不一定是函数.

### 原型链
JS是单继承的,当试图获取一个对象的某个属性时,如果对象本身没有这个属性,那么就会通过`__proto__`去它的构造函数的原型对象`prototype`中查找,因为`prototype`也是一个对象,如果`prototype`中也不存在这个属性,那么它将通过`__proto__`去查找,直到查找到`null`(避免死循环),而这整个流程就被称作原型链.`Object.prototype`是原型链的最顶端,所有对象从它继承包括`toString`和`valueOf`等方法和属性.它的原型指向`null`.
![原型链](https://www.laruence.com/medias/2010/05/javascript_object_layout.jpg)

### 原型链继承

使用父类的实例重写子类的原型对象
```
function Animal(){
  this.species = 'animal'
}
Animal.prototype.getInfo = function(){
  console.log(this.species,'-----------species')
}
function Dog(){
  this.species = 'dog'
}
Dog.prototype = new Animal()
```
优点: 可以复用父类的方法
缺点: 父类的`prototype`中的属性和方法会被所有子类共享,更改任何一个属性,所有子类都会受到影响;子类的实例不能给父类的构造函数传递参数.

### 借用构造函数继承

在子类的构造函数内部通过`call`或者`apply`改变父类构造函数的`this`指向,将父类的属性和方法重写个到子类上.
```
function Animal(){
  this.species = 'animal'
}
Animal.prototype.getInfo = function(){
  console.log(this.species, '-------species')
}
function Dog(){
  Animal.call(this)
  this.species = 'dog'
}
```
优点:可以在子类构造函数中向父类构造函数中传递参数;父类的`prototype`对象不会被共享,避免了原型链继承的缺点.
缺点:由于父类的`prototype`对象不会被共享,那么子类也不能访问父类原型上的方法,除非将所有方法写入构造函数中

### 组合继承(组合了原型链继承和构造函数继承)

综合原型链继承和借用构造函数继承两种方法,通过原型链继承共享原型上的属性和方法,通过构造函数继承实例属性,这样既可以把方法定义在原型上实现共享,又可以让每个实例拥有自己的属性.
```
function Animal(){
  this.species = 'animal'
}
Animal.prototype.getInfo = function(){
  console.log(this.species, '---------species')
}
function Dog(){
  Animal.call(this)
  this.species = 'dog'
}
Dog.prototype = new Animal()
Dog.prototype.constructor = Dog
```
优点: 父类的`prototype`中的属性和方法,子类都可以共享;可以通过子类构造函数向父类构造函数传参;父类构造函数中的属性不会被共享.
缺点:调用了两次父类构造函数:创建子类原型的时候和在子类构造函数内部,会生成两份实例.

### 原型式继承

对父类的属性和方法进行浅复制

```
function copyObj(obj){
  function Fn(){}
  Fn.prototype = obj
  return new Fn()
}
var animail = {
  species: 'animal'
}
var dog = copyObj(animal)
```
优点:可以复用父类的方法
缺点:父类的引用会被所有子类共享;子类实例不能向父类传参
当`Object.create`只有一个参数时,效果与`copyObj`相同;`new`与`Object.create`的区别:`new`创建一个对象,并执行构造函数;`Object.create`创造一个对象,但不执行构造函数

### 寄生式继承

在原型式继承的基础上加强继承的能力

```
function copyObj(obj){
  function Fn(){}
  Fn.prototype = obj
  return new Fn()
}
function inheriObj(obj){
  var subobj = copyObj(obj)
  subobj.getInfo = function(){
    console.log(this.species,'-----species')
  }
  return subobj
}
```

### 寄生式组合继承(最佳模式)

通过借用构造函数继承属性,通过原型链混成的方式继承构造方法

```
function copyObj(obj){
  function Fn(){}
  Fn.prototype = obj
  return new Fn()
}
function inheritObj(subObj,supObj){
  var prototype = copyObj(supObj)
  prototype.constructor = subObj;
  subObj.prototype = prototype
}

function Animal(){
  this.species = 'animal'
}
Animal.prototype.getInfo = function(){
  console.log(this.species,'------species')
}
function Dog(){
  Animal.call(this)
  this.species = 'dog'
}
inheritObj(Dog,Animal)
```
优点:只调用了一次父类构造函数;父类方法可以复用;父类的引用属性不会被共享;子类构造函数可以向父类构造函数传递参数.

### ES6 Class

本质上是寄生组合式继承的语法糖

```
class Animal{
  constructor(){
    this.species = 'animal'
  }
  getInfo(){
    console.log(this.species, '-----animal')
  }
}

class Dog extends Animal{
  constructor(){
    super()
    this.species = 'dog'
  }
}
```
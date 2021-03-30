---
title: valueOf/toString
date: 2021-03-29 22:01:32
tags:
---

JS是弱类型语言,这意味着在JS中没有确定类型的变量,其类型只取决于当时的值的类型.当我们进行操作的时候,有时候需要用到一些类型变换.这其中就分为显示转换和隐士转换.JS的数据类型有8种,`Number`、`String`、`Boolean`、`undefined`、`Object`、`Null`、`bigInt`、`Symbol`,其中基础类型(原始值):`Number`、`String`、`Boolean`、`undefined`、`Null`、`bigInt`、`Symbol`,复杂类型(对象值):`Object(Function、Array、Date、RegEx等)`

### 显示转换

#### Number

- Number: 严格转换利用`Number`强制转换为数字`ToNumber`
- parseInt:从左到右依此转换,能转换就转换不能转换就停止;如果第一个就不能转换返回`NaN`;转换为整数
- parseFloat:从左到右依此转换,能转换就转换不能转换就停止;如果第一个就不能转换返回`NaN`;可以转换小数
- Math.round: 四舍五入,如果出现非数字字符,返回`NaN`

#### String

- String: 严格转换,利用`String`强制转换为字符串`ToString`
- toString: JS提供的`toString`方法转换字符串

### Boolean

- Boolean: 严格转换,利用`Boolean`强制转换为布尔值

### 隐士转换

隐士转换一般发生在使用操作符时,其中尤以`+`、`==`引起的问题居多,其他操作符一般都针对`number`类型(将运算的值转为number类型即可).JS隐士转换主要分为:`ToPrimitive(转为原始值)`、`ToNumber(转为数字)`、`ToString(转为字符串)`.

#### ToPrimitive

JS引擎提供`ToPrimitive(input,preferredType?)`通过一个必传参数`input`和一个可选参数`preferedType(Number或String)`来将`input`转换成一个原始值或者报错.当`input`为一个原始值的类型时,那么不需要参考`perferredType`直接就会返回这个值.但是如果`input`是一个对象值呢? 除了`Date`类型的对象`preferedType`被设置为`String`(`Date`对象更期望获取转换后的字符串),其余的都会被设置为`Number`.

##### perferredType=Number

1. 调用该对象的`valueOf()`方法,如果`valueOf()`返回的是一个原始值,则返回这个原始值
2. 调用该对象的`toString()`方法,如果`toString()`返回的是一个原始值,则返回这个原始值
3. 否则,抛出`TypeError`异常

##### perferredType=String

1. 调用该对象的`toString()`方法,如果`toString()`返回的是一个原始值,则返回这个原始值
2. 调用该对象的`valueOf()`方法,如果`valueOf()`返回的是一个原始值,则返回这个原始值
3. 否则,抛出`TypeError`异常

`Object.prototype`处于原型链的最顶端,它的`toString()`和`valueOf()`方法都会被`Object`的派生对象所继承,当然也可以更改.

##### valueOf

- `Number|String|Boolean => number|string|boolean`,即`Number`、`String`、`Boolean`类型的对象通过`valueOf()`转换成各自的原始值,即`Number => num; String=>str; Boolean => bool`
- `Date => number`,`Date`类型的对象通过`valueOf()`转换成数字`Date => num`
- 其余对象返回的均为`this`,即对象本身.

##### toString

- `Number|Boolean|String|Array|Date|RegExp|Function`这几种构造函数生成的对象,通过`toString()`转换成相应的字符串形式.(自己封装的`toString`方法)即`Number => 'num'; Boolean => 'bool'; Array => arr.join(,);String => 'str';Function => JS源码字符串; Date => 可读时间字符串;RgeExp => 'regexp'`
- 其他类型的对象返回该对象的类型(继承的`Object.prototype`)`Object => [object Object]`
- `Null,Undefined`调用报错

#### ToNumber

`ToNumber(input)`将输入的值转换为数字

- Undefined: `Undefined => NaN`
- Null: `Null => +0`
- Boolean: `true => 1 fale => 0`
- Number: 结果等于输入的参数
- String: `str => Number(str); Number('') => 0; Number(str) => NaN`
- Object: `obj => ToPrimitive(obj,Number)`

#### ToString

`ToString(input)`将输入的值转换为数字

- Undefined: `Undefined => 'undefined'`
- Null: `Null => 'null'`
- Boolean: `true => 'true' false => 'false'`
- String: 结果等于输入的参数
- Number:`num => 'num'`(一般情况)
- Object: `obj => ToPrimitive(obj,String)`

#### 转换规则 

涉及`+`、`==`引起的隐式转换主要根据运算符的执行顺序和优先级来决定,这里`+`和`==`都是按照从左到右来计算

```js
{} + {} => '[object Object][object Object]'
```

1.  `+`号 运算符可以进行数字和字符串的计算, 那么就需要将`{}`转换成原始值
2.  默认对象(除Date)都是`ToPrimitive(input,Number)`,则通过调用`valueOf() => this`得到的还是`{}`,而非原始值,依然不能进行操作计算
3.  则调用`toString() => [object Object]`,得到的是原始值,可以进行操作
4. 获得`[object Object][object Object]`

```js
2 * {} => NaN
```

1. `*` 只能针对`num`原始值计算,那么就需要将`{}`转换为`num`原始值
2. `{}`默认`ToPrimitive(input,Number)`,调用`valueOf() => this`,获取`{}`,依然不是原始值
3. 调用`toString() => [object Object]`,此时已经是原始值,但是仍然不是`num`,那么就需要按照ToString 将`[object Object]`转换为数字`NaN`
4. `2 * NaN => NaN`

```js
[] == ![] => true
```

1. 首先将`[]`转化为原始值,`ToPrimitive(input,Number)`调用`valueOf() => this`,得到`[]`依然不是原始值,不能操作
2. 调用`toString() => ''`,获得原始值
3. `[]`转换为布尔值获取`true`,`![]`执行逻辑非操作,返回布尔值,`![] => false`
4. 此时`'' == false`不能进行比较就需要将他们转为数字`0 == 0 => true`

```js
const a = {
  i : 1,
  toString(){
    return this.i ++
  }
}
a == 1 && a == 2 && a == 3 => true
```

1. 定义了对象a和重写了`toString`方法
2. `a == 1`时,a不是一个原始值`ToPrimitive(input,Number)`,调用`valueOf() => this`,得到a对象不是原始值
3. 于是调用`toString()`,因为`a`重写了`toString`方法,在`toString`时执行`this.i ++`操作返回`this.i => 1`,同时`this.i +1 = 2`,因此得到原始值1,`a == 1 => true`
4. 同理`a == 2 => true; a == 3 => true` => `true`

#### == 和 +

- `x == y`
  - 同类型:原始值,当 x 和 y,直接比较,`NaN !== NaN === Object.is(NaN,NaN)`;对象值, x和y是同一引用时相等,否则不相等
  - 不同类型: x 和 y中存在`String`则转换为`ToNumber`;如果存在`Boolean`则转换为`ToNumber`;如果存在`Object`,则通过`ToPrimitive`转换为原始值,再通过`ToNumber`进行比较
- `x + y`
  - 原始值:x 和 y中存在`String`类型则通过`ToString`转换后操作;否则通过`ToNumber`转换后操作
  - 对象值:通过`ToPrimitive`后转换成原始值进行操作


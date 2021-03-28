---
title: Apply/Call/Bind
date: 2021-03-28 10:38:05
tags:
---

`Apply/Call/Bind`的作用就是改变函数运行时的`this`指向

### Apply

`Function.prototype.apply(context,[args])`通过接收两个参数,第一个参数是`this`的指向,如果`context`为`null`则指向`window`,第二个参数为一个数组或类数组的对象(传入函数的参数)

```js
Function.prototype.apply = function(context){
  if(typeof this !== 'function'){
    throw new Error('必须是一个函数')
  }
  // 创建对象,如果context不存在则指向window
  context = context ? Object(context) : Window
  // 将当前方法挂载到context对象上
  context.fn = this
  // 获取传入当前方法的参数(一个数组或类数组)
  const args = [...argumenus][1]
  let res;
  // 获取函数执行结果
  if(args){
    res = context.fn(args)
  }
  res = context.fn()
  // 删除对象上fn属性
  delete context.fn;
  // 执行返回
  return res
}
```

### Call

`Function.prototype.call(context,arg1,arg2,...)`通过接收多个参数,第一个参数是`this`的指向,如果`context`为`null`则指向`window`,其余参数为传入函数的参数

```js
Function.prototype.call = function(context){
  if(typeof this !== 'function'){
    throw new Error('必须是一个函数')
  }
  // 创建对象,如果context不存在则指向window
  context = context ? Object(context) : window
  // 将当前方法挂载到context对象上.
  context.fn = this
  // 获取要传入方法的参数
  const args = [...arguments].slice(1)
  let res;
  // 获取函数执行结果
  if(args){
    res = context.fn(args)
  }
  res = context.fn()
  // 删除对象上的fn属性
  delete context.fn();
  // 执行返回
  return res
}
```

### Bind

`Function.prototype.bind(context,arg1,arg2,...)`通过传入多个参数,第一个参数为`this`指向的对象,其余参数为函数的传入参数,并返回一个新的函数

```js
Function.prototype.bind = function(context){
  if(typeof this !== 'function'){
    throw new Error('必须是一个函数')
  }
  // 创建对象,如果context不存在则指向window
  context = context ? Object(context) : window
  // 定义self为当前函数
  const fn = this
  // 获取要传入的参数
  const args = [...arguments].slice(1)

  // 定义返回的函数
  function fBound(){
    // 获取调用bind返回新函数的参数
    const args1 = [...arguments].slice(1)
    // 合并参数
    const argslist = args.concat(args1)
    if(this instanceof fBound){
      // 当返回的函数作为构造函数,new操作时this指向不改变
      fn.apply(this,argslist)
    }else{
      fn.apply(context,argslist)
    }
  }
  // 通过原型继承,以原型链的方式,将返回函数的原型指向fn
  functoin O(){}
  O.prototype = fn.prototye
  fBound.prototype = new O()
  return fBound
}
```

#### new实现

```js
const obj = {}
obj.__proto__ = Func.prototype
const res = Func.call(obj,args)
return typeof res === 'object' ? res : obj
```
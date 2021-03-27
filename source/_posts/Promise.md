---
title: Promise
date: 2021-03-26 21:03:16
tags:
---

> `promise`表示一个异步操作的最终结果.和一个`promise`进行交互的主要方式是通过它的`then`方法,该方法注册了两个回调用来接收成功后的响应和失败的原因.
`Promise`必须提供一个`then`方法来访问当前或者最终的返回(可能被访问多次),同时返回一个`Promise`,它接受两个参数`Promise.then(onFulfilled,onRejected)`,它们必须是函数,否则将会被忽略.`onFulfilled`必须在`fulfilled`调用(只能调用一次),`onRejected`必须在`rejected`调用(只能调用一次).

### 1. Promise状态

> 一个`Promise`的当前状态只能是`pending`、`fulfilled`和`rejected`三种之一,当一个`promise`处于等待状态时,可能会变为`fulfilled`或者`rejected`状态(不可逆),但是当处于`fulfilled`或`rejected`状态时,状态不可更改.

```
// 定义promise的状态
const pending = 'pending'
const fulfilled = 'fulfilled'
const rejected = 'rejected'

class Promise{
  constructor(executor){
    // executor必须是个函数
    if(typeof executor !== 'function') throw new Error('executor must be a function')
    // 定义初始化的状态
    this.status = pending
    // 定义promise的成功值
    this.value = undefined
    // 定义promise失败的原因
    this.reason = undefined
    // 立即执行excutor函数解决new 实例出来的promise状态(可以添加逻辑判断)
    try{
      executor(this._resolve.bind(this),this._reject.bind(this))
    }catch(e){
      this._reject(e)
    }
  }
  // 执行成功,resolve改变状态,赋值响应
  _resolve(val){
    if(this.status === pending){
      this.status = fulfilled
      this.value = val
    }
  }
  // 执行失败,reject改变状态,赋值失败原因
  _reject(val){
    if(this.status === pending){
      this.status = rejected
      this.reason = val
    }
  }
}
```

### 2. Promise.then

> `Promise`必须提供一个`then`方法来访问当前或最终的响应或者失败的原因.`Promise.then(onFulfilled,onRejected)`,`onFulfilled`和`onRejected`如果不是函数将会被忽略,`onFulfilled`必须在`Promise.resolve`后才会被调用,且`promise`的值作为它的第一个参数;`onRejected`必须在`Promise.reject`后被调用,且`promise`的原因作为它的第一个参数.它们都只能被调用一次.同一个`promise`上`then`可能会被调用多次,如果`resolve`,那么`onFulfilled`会按照调用`then`的顺序执行,如果`reject`那么`onReject`也会按照调用`then`的顺序执行

```
class Promise{
  constructor(executor){
    ...
    // 添加成功回调函数数组,存储pending状态下then中的onFulfilled
    this.fulfillQueues = []
    // 添加失败回调函数数组,存储pending状态下then中的onRejected
    this.rejectQueues = []
  }
  _resolve(val){
    if(this.status === pending){
      this.statsu = fulFilled
      this.value = val
      // resolve 按照then的调用顺序执行onFulfilled
      this.fulfillQueues.forEach((fn) => fn(val))
    }
  }
  _reject(val){
    if(this.ststus === pending){
      this.status = rejected
      this.reason = val
      // reject 按照then的调用顺序执行onRejected
      this.rejectQueues.forEach((fn) => fn(val))
    }
  }
  then(onFulfilled,onRejected){
    const {value,reason,status} = this
    swicth(status){
      // pending状态下,将onfulfiled和onRejected存入数组等待promise状态改变
      case 'pending':
        this.fulfillQueues.push(onFulfilled)
        this.rejectQueues.push(onRejected)
        break;
      // fulFilled执行
      case 'fulFilled':
        onFulFilled(value)
        break;
        // rejected执行
      case 'rejected'
         onRejected(reason)
         break;
    }
  }
}

```

### 3. 链式调用

> `then(onFulfilled,onRejected)`必须返回一个`promise`

```
promise2 = promise1.then(onFulfilled,onRejected)
```
> 如果`onFulfilled`或者`onRejected`返回一个值`x`,运行promise解决程序`[[resolve]](promise2,x)`;如果`onFulfilled`或者`onRejected`抛出一个异常`e`,`promise2`必须以`e`作为原因`reject`;如果`onFulfilled`不是一个函数且`promise1`被`resolve`,`promise2`与`promise1`必须以相同的值被`resolve`;如果`onRejected`不是一个函数且`promise1`被`reject`,`promise2`与`promise1`必须以相同的原因被`reject`

```
class Promise{
  constructor(executor){
    ...
  }
  ...
  then(onFulfilled,onRejected){
    const {status,value,reason} = this
    const promise2 = new Promise((fulfillNext,rejectNext) => {
      // 定义一个成功执行的方法
      const fulFn = (value) => {
        try{
          // 如果onFulfilled不是函数,调用新的promise对象的resolve
          if(typeof onFulfilled !== 'function'){
            fulfillNext(value)
          }else{
            const res = onFulfilled(value)
            resolvePromise(promise2,res,fulFillNext,rejectNext)
          }
        }catch(err){
          rejectNext(err)
        }
      }
      // 定义一个失败执行的方法
      const rejFn = (err) => {
        try{
          // 如果onRejected不是函数,调用新的promise对象的reject
          if(typeof onRejected !== 'function'){
            rejectNext(err)
          }else {
            const res = onRejected(err)
            resolvePromise(promise2,res,fulfillNext,rejectNext)
          }
        }catch(err){
          rejectNext(err)
        }
      }
      switch(status){
        case 'pending':
         this.fulfillQueues.push(fulFn)
         this.rejectQueues.push(rejFn)
         break
        case 'fulFilled':
          fulFn(value)
        break;
        case 'rejected':
          rejFn(reason)
        break;
      }      
    })
    return promise2
  }
}
```

### 4. Promise解决程序

> Promise解决程序是一个抽象操作,它以一个`promise`和一个值传入,我们将其表示为`[[resolve]](promise,x)`,如果`x`是一个具有`then`方法的对象或函数,就尝试采用`x`的status并假设`x`从某种程度上类似于`promise`,否则就是用`x`作为`promise.resolve`.这就使得`Promise`的实现更具有通用性:只要其暴露一个遵循`Promise/A+`协议的`then`方法即可.
> 如果`promise`和`x`是同一个对象,那么抛出`TypeError`来`reject promise`.否则`x`是一个`promise`:当`pending`时,`promise`保持`pending`直到`x`状态改变;当`fulfilled`时,`promise`以与`x`相同的值resolve;当`reject`时,`promise`以与`x`相同的值reject.否则`x`是一个对象或函数时:让`then`变为`x.then`;如果`x.then`抛出异常`e`,则将`e`作为原因reject promise.调用`x.then(resolvePromise,rejectPromise)`,通过传入`resolvePromise`或者`rejectPromise`来`resolve`或`reject`promise,如果;两个参数同时或多次调用以第一次调用优先,`then`调用抛出异常时,如果`rejectPromise`没被调用则使用`rejectPromise`,否则直接`reject`promise.如果`x`不是对象或者函数 或者 `then`不是函数则以`x`resolve`promise`

```
class Promise{
  constructor(executor){
    ...
  }
  ...
  resolvePromise(promise,x,resolve,reject){
    if(promise === x){
      throw new Error('TypeError')
    }
    if(x && typeof x === 'object' || typeof x === 'function'){
      // 定义当前回调是否被调用
      let used;
      try{
        const then = x.then
        if(typeof then === 'function'){
          x.then(y => {
            if(used) return
            used = true
            resolvePromise(promise,y,resolve,reject)
          },(e) => {
            if(used) return 
            used = true
            reject(e)
          })
        }else{
          if(used) return
          used = true
          resolve(x)
        }
      }catch(err){
        if(used) return
        used = true
        reject(err)
      }
    }else{
      resolve(x)
    }
  }
}
```

因此可得出
```
class Promise{
  constructor(executor){
    ...
  }
  ...
  then(onFulfilled,onRejected){
    const {status,value,reason} = this
    ...
    // 返回一个`promise`对象
    return new Promise((fulfillNext,rejectNext) => {
      // 定义一个成功执行的方法
      const fulFn = (value) => {
        try{
          // 如果onFulfilled不是函数,调用新的promise对象的resolve
          if(typeof onFulfilled !== 'function'){
            fulfillNext(value)
          }else{
            判断 onFulfilled的返回是不是一个promise对象
            const res = onFulfilled(value)
            // 执行then方法来返回最终响应
            if(res instance of Promise){
              res.then(fulfillNext,rejectNext)
            }else{
              // 否则resolve输出
              fulfillNext(res)
            }
          }
        }catch(err){
          rejectNext(err)
        }
      }
      // 定义一个失败执行的方法
      const rejFn = (err) => {
        try{
          // 如果onRejected不是函数,调用新的promise对象的reject
          if(typeof onRejected !== 'function'){
            rejectNext(err)
          }else {
            // 判断onRejected的返回是不是一个promise对象
            const res = onRejected(err)
            // 执行then方法返回响应
            if(res instanceof Promise){
              res.then(fulfillNext,rejectNext)
            }else{
              //否则reject输出
              fullfillNext(res)
            }
          }
        }catch(err){
          rejectNext(err)
        }
      }
      switch(status){
        case 'pending':
         this.fulfillQueues.push(fulFn)
         this.rejectQueues.push(rejFn)
         break
        case 'fulFilled':
          fulFn(value)
        break;
        case 'rejected':
          rejFn(reason)
        break;
      }
    })
  }
}
```

### 5. resolve和reject的参数

我们还需要考虑到resolve和reject的参数,通常我们的处理只是一个变量对象,假如是一个`promise`对象呢

```
_resolve(val){
  cost run = () => {
    if(this.status === pending) return 
    this.staus = fulfilled
    const fulFn = (val) => {
      let cb;
      while(cb = this.fulFillQueues.shift()){
        cb(val)
      }
    }
    const rejFn = (err) => {
      let cb;
      while(cb = this.rejectQueues.shift()){
        cb(err)
      }
    }
    if (val instanceof Promise){
      val.then((value) => {
        this.value = value
        fulFn(value)
      },err => rejFn(err))
    }else{
      this.value = value
      fulFn(value)
    }
  }
  // 支持同步Promise,多个Promise执行宏任务加入到任务队列
  setTimeout(run,0)
}
```
### 6. Promise的其他方法封装

- resolve

```
resolve(val){
  if(val instanceof Promise) return val
  return new Promise(resolve => resolve(val))
}
```

- reject

```
reject(val){
  if(val instanceof Promise) return val
  return new Promise((resolve,reject) => reject(val))
}
```

- catch

```
catch(onRejected){
  this.then(undefined,onRejected)
}
```

- all

```
all(list){
  return new Promise((resolve,reject) => {
    let reslist = []
    for( let p of list){
      this.resolve(p).then((res) => {
        reslist.push(res)
      },err => {
        reject(err)
      })
    }
    if(reslist.length === list.length){
      resolve(reslist)
    }
  })
}
```

- race

```
race(list){
  return new Promise((resolve,reject) => {
    for(let p of list){
      this.resolve(p).then(res => {
        resolve(res)
      },err => reject(err))
    }
  })
}
```

- finally

```
finally(cb){
  return this.then(
    value => Promise.resolve(cb()).then(() => value),
    reason => Promise.resolve(cb()).then(() => throw new Error(reason))
  )
}
```

### 最终产出
```
const pending = 'pending'
const fulfilled = 'fulfilled'
const rejected = 'rejected'

class Promise{
  constructor(executor){
    if(typeof executor!== 'function') throw new Error('executor must be a function')
    this.status = 'pending'
    this.value = undefined
    this.reason = undefined
    this.fulfillQueues = []
    this.rejectQueues = []
    try{
      executor(this._resolve.bind(this),this._reject.bind(this))
    }catch(err){
      this._reject(err)
    }
  }

  _resolve(val){
    const run = () => {
      if(this.status !== pending) return 
      this.status = fulfilled
      const fulFn = (val) =>{
        let cb;
        while(cb = this.fulFullQueues.shift()){
          cb(val)
        }
      }
      const rejFn = (err) => {
        let cb;
        while(cb = this.rejectQueues.shift()){
          cb(err)
        }
      }
      if(val instanceof Promise){
        val.then(value=>{
          this.value = value
          fulFn(value)
        },err => rejFn(err))
      }else{
        this.value = value
        fulFn(value)
      }
    }
    setTimeout(run,0)
  }
  _reject(err){
    const run = () => {
      if(this.status !== pending) return 
      this.status = rejected
      const fulFn = (val) => {
        let cb;
        while(cb = this.fulFullQueues.shift()){
          cb(val)
        }
      }
      const rejFn = (err) =>{
        let cb;
        while(cb = this.rejectQueues.shift()){
          cb(err)
        }
      }
      if(err instanceof Promise){
        err.then((value) => fulFn(value),err => {
          this.reason = err
          rejFn(err)
        })
      }else{
        this.reason = err
        rejFn(err)
      }
    }
    setTimeout(run,0)
  }
  then(onFulFill,onReject){
    return new Promise((fulNext,rejNext) => {
      const fulFn = (value) => {
        try{
          if(typeof onFulFill !== 'function'){
            fulNext(value)
          }else{
            const res = onFulFill(value)
            if(res instanceof Promise){
              res.then(fulNext,rejNext)
            }else{
              fulNext(value)
            }
          }
        }catch(err){
          rejNext(err)
        }
      }
      const rejFn = (err) => {
        try{
          if(typeof onReject !== 'function'){
            rejNext(err)
          }else{
            const res = onReject(err)
            if(res instanceof Promise){
              res.then(fulNext,rejNext)
            }else{
              fulNext(err)
            }
          }
        }catch(err){
          rejNext(err)
        }
        const {status,value,reason} = this
        switch(status){
          case 'pending':
            this.fulFullQueues.push(fulFn)
            this.rejectQueues.push(rejFn)
            break;
          case 'fulfilled':
            fulFn(value)
            break;
          case 'rejected':
            rejFn(value)
            break;
        }
      }
    })
  }
  resolve(val){
    if(val instanceof Promise) return val
    return new Promise((resolve) => resolve(val))
  }
  reject(err){
    if(err instanceof Promise) return err
    return new Promise((resolve,reject) => reject(err))
  }
  catch(cb){
    return this.then(undefined,cb)
  }
  all(list){
    return new Promise((resolve,reject) => {
      const reslist = []
      for(let p of list){
        this.resolve(p).then(value => reslist.push(value),err => reject(err))
      }
      if(reslist.length === list.length) resolve(reslist)
    })
  }
  race(list){
    return new Promise((resolve,reject) => {
      for(let p of list){
        this.resolve(p).then(value => resolve(value),err=> reject(err))
      }
    })
  }
  finally(cb){
    return this.then(
      value => Promise.resolve(cb()).then(() => value)
      err => Promise.resolve(cb()).then(() => throw new Error(err))
    )
  }
}
```
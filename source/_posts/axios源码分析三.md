---
title: axios源码分析三
date: 2021-07-01 21:47:16
tags:
---

### dispatchRequest

```js
promis = dispatchRequest(newConfig)
```

#### dispatchRequest

```js
function dispatchRequest(config){
  throwIfCancellationRequested(config)
  config.headers = config.headers || {}
  // 根据请求类型输出请求的data转化
  config.data = transfromData.call(config,config.data,config.headers,config.transformRequest)
  // 根据请求类型输出header
  config.headers = utils.merge(config.headers.common || {},config.headers[config.method] || {},config.headers)
  // 移除不必要的header
  utils.forEach(['delete','get','head','post','put','patch','common'],(method) => {
    delete config.headers[method]
  })
  // 获取http请求方法
  var adapter = config.adapter || defaults.adapter
  // 进行数据请求
  return adapter(config).then((response) => {
    // 判断是否cancelToken进行请求终止
    throwIfCancellationRequested(config)
    // 根据transfromResponse对响应进行转化
    response.data = transformData.call(config,response.data,response.headers,config.transformResponse)
  },(reason) => {
    if(!isCancel(reason)){
      throwIfCancellationRequested(config)
      if(reason&& reason.respobse){
        // 判断reason数据进行转化
        reason.response.data = transfromData.call(config,reason.response.data,reason.response.headers,config.transfromResponse)
      }
    }
    return Promise.reject(reason)
  })
}
```

##### throwIfCancellationRequested

```js
function throwIfCancellationRequested(config){
  //  如果出现请求报错，throw异常
  if(config.cancelToken){
    config.cancelToken.throwIfRequested()
  }
}
```

##### transformData

```js
function transformData(data,headers,fns){
  var context = this || defaults
  // 遍历fn,输出对data的转化
  utils.forEach(fns,(fn) => {
    data = fn.call(context,data,headers)
  })
  return data
}
```

### Cancel

```js
axios.Cancel = require('./cancel/Cancel')
axios.CancelToken = require('./cancel/CancelToken')
axios.isCancel = require('./cancel/isCancel')
```

#### Cancel

```js
// 声明cancel类
function Cancel(message){
  this.message = message
}
Cancel.prototype.toString = function (){
  return 'Cancel' + (this.message ? ':' + this.message : '')
}
Cancel.prototype.__CANCEL__ = true
```

#### CancelToken

```js
function CancelToken(executor){
  if(typeof executor !== 'function'){
    throw new Error('executor must be a function')
  }
  var resolvePromise
  // 获取resolve
  this.promise = new Promise((resolve)=>{
    resolvePromise = resolve
  })
  var token = this
  // 执行生成reason
  executor(function(message){
    if(token.reason){
      return
    }
    token.reason = new Cancel(message)
    resolvePromise(token.reason)
  })
}
CancelToken.prototype.throwIfRequested = function(){
  if(this.reason){
    throw this.reason
  }
}
CancelTokem.source = function(){
  var cancel
  var token = new CancelToken(function executor(c){
    cancel = c
  })
  return {token,cancel}
}
```

#### isCancel

```js
// 判断是否cancel
function isCancel(value){
  return !!(value && value.__CANCEL__)
}
```
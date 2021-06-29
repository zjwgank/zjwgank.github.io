---
title: axios源码分析一
date: 2021-06-28 22:25:53
tags:
---
### 入口

```js
// package.json
{
  ...
  main: 'index.js',
  ...
}
// index.js
module.exports = require('./lib/axios')
```

### axios

```js
var utils = require('./utils')
var bind = require('./helpers/bind')
var Axios = require('./core/Axios')
var mergeConfig = require('./core/mergeConfig')
var defaults = require('./defaults')
// 依据方法创建axios实例
var axios = createInstance(defaults)
axios.Axios = Axios
// create方法创建Axios实例
axios.create = function create(instanceConfig){
  return createInstance(mergeConfig(axios.defaults,instanceConfig))
}
// 扩展Cancel/CancelToken/isCancel
axios.Cancel = require('./cancel/Cancel')
axios.CancelToken = require('./cancel/CancelToken')
axios.isCancel = require('./cancel/isCancel')
// 处理并发请求
axios.all = function all(promises){
  return Promise.all(promises)
}
axios.spread = require('./helpers/spread')
axios.isAxiosError = require('./helpers/isAxiosError')
module.exports = axios
module.exports.default = axios
```

#### createInstance

```js
function createInstance(defaultConfig){
  // 根据配置实例化Axios对象
  var context = new Axios(defaultConfig)
  // Axios.prototype.request的this指向Axios实例
  var instance = bind(Axios.prototype.request,context)
  // 复制Axios.prototype的属性到Axios实例上并this指向
  utils.extend(instance,Axios.prototype,context)
  // 复制Axios对象的属性到request上
  utils.extend(instance,context)
  return instance
}
```

##### bind 

```js
function bind(fn,thisArg){
  // 闭包返回，绑定this指向
  return function wrap(){
    var args = new Array(arguments.length)
    for(var i = 0; i < args.length; i ++){
      args[i] = arguments[i]
    }
    return fn.apply(thisArg,args)
  }
}
```

##### util.extend 

```js
//  forEach遍历
function extend(a,b,thisArg){
  forEach(b,function assignValue(val,key) {
    // 扩展fuctionVal,绑定this
    if(thisArg && typeof val === 'function'){
      a[key] = bind(val,thisArg)
    }else{
      a[key] = val
    }
  })
  return a
}
```

#### defaults

```js
// 定义默认数据传输格式
var DEFAULT_CONTENT_TYPE = {'Content-Type':'application/x-www-form-urlencoded'}
// 默认axios配置
var defaults = {
  transitional:{
    silentJSONParsing:true, // 忽略JSON.parse(response.body)的错误
    forcedJSONParsing:true, // 当responseType!== json时将response转化为json
    clarifyTimeoutError:false, // 当请求超时返回ETIMEDOUT而不是ECONNABORTED
  },
  adapter:getDefaultAdapter(), // 自定义请求
  // 在request到达server前自定义请求发送的数据
  transformRequest:[function transformRequest(data,headers){
    normalizeHeaderName(headers,'Accept')
    normalizeHeaderName(headers,'Content-Type')
    // 针对各种请求数据的类型的处理
    if(utils.isFormData(data) || utils.isArrayBuffer(data) || utils.isBuffer(data) || utils.isStream(data) || utils.isFile(data) || utils.isBlob(data)){
      return data
    }
    if(utils.isArrayBufferView(data)){
      return data.buffer
    }
    if(utils.isURLSearchParams(data)){
      setContentTypeIfUnset(headers,'application/x-www-form-urlencoded;charset=utf-8')
      return data.toString()
    }
    if(utils.isObject(data) || (header && headers['Content-Type'] === 'application/json')){
      SetContentTypeIfUnset(headers,'applicaion/json')
      return JSON.stringify(data)
    }
    return data
  }],
  // .then/catch前处理response
  transformResponse:[function transformResponse(data){
    const transitional = this.transitional
    // 是否忽略JSON.Parse的错误
    var slientJSONParsing = transitional && transitional.silentJSONParsing
    // 是否强制将res转化为json
    var forcedJSONParsing = transitional && transitional.forcedJSONParsing
    var strictJSONParsing = !silentJSONParsing && this.responseType === 'json'
    if(strictJSONParsing || (forcedJSONParsing && utils.istring(data) && data.length)){
      try{
        return JSON.parse(data)
      }catch(e){
        if(strictJSONParsing){
          if(e.name === 'SyntaxError'){
            throw enhanceError(e,this,'E_JSON_PARSE')
          }
          throw e
        }
      }
    }
    return data
  }],
  timeout: 0, // 请求超市时间
  xsrfCookieName:'XSRF-TOKEN', // 用来校验xsrf请求的cookieName
  xsrfHeaderName:'X-XSRF-TOKEN', // 用来校验xsrf请求的headerName
  maxContentLength: -1, // 响应体的最大字符
  maxBodyLength: -1, // 请求体的最大字符
  validateStatus:function validateStatus(status){
    return status >= 200 && status < 300 // 校验请求成功状态
  }
}
defaults.headers = {
  common:{'Accept':'application/json,text/plain,*/*'}
}
// 设置get/delete/head/post/put/patch默认的header头的Content-Type
utils.forEach(['delete','get','head'],function forEachMethodNoData(method){
  defaults.headers[method] = {}
})
utils.forEach(['post','put','patch'],function forEachMethodWithData(method){
  defaults.headers[method] = utils.merge(DEFAULT_CONTENT_TYPE)
})
```

##### getDefaultAdapter

```js
// 自定义请求
function getDefaultAdapter(){
  var adapter
  // 判断当前处于浏览器环境还是node环境返回
  if(typeof XMLHttpRequest !== 'undefined'){
    adapter = require('./adapter/xhr')
  }else if(typeof process !== 'undefined' && Object.prototype.toString.call(process) === '[object process]'){
    adapter = require('./adapter/http')
  }
  return adapter
}
```

##### setContentTypeIfUnset

```js
// 设置header的Content-Type
function setContentTypeIfUnset(headers,value){
  if(!utils.isUndefined(headers) && utils.isUndefined(headers['Content-Type'])){
    headers['Content-Type'] = value
  }
}
```

##### normalizeHeaderName

```js
// 清除headers中的小写设置
function normalizeHeaderName(headers,normalizedName){
  utils.forEach(headers,function processHeader(value,name){
    if(name !== normalizedName && name.toUpperCase() === normalizedName.toUpperCase()){
      headers[normlizedName] = value
      delete headers[name]
    }
  })
}
```

##### merge

```js
function merge(){
  var result = {}
  // 合并数据
  function assignValue(val,key){
    if(isPlainObject(result[key]) && isPlainObject(val)){
      result[key] = merge(result[key],val)
    }else if(isPlainObject(val)){
      result[key] = merge({},val)
    }else if(isArray(val)){
      result[key] = val.slice()
    }else{
      result[key] = val
    }
  }
  // 遍历参数 
  for(var i = 0,l = arguments.length;i < l;i ++){
    forEach(arguments[i],assignValue)
  }
}
```

##### enhanceError

```js
// 生成axios的错误信息
function enhanceError(error,config,code,request,response){
  error.config = config
  if(code){
    error.code = code
  }
  error.request = request
  error.response = response
  error.isAxiosError = true
  error.toJSON = function toJSON(){
    return {
      message: this.message,
      name:this.name,
      description: this.description,
      number: this.number,
      fileName: this.fileName,
      lineNumber: this.lineNumber,
      columnNumber: this.columnNumber,
      stack:this.stack,
      config: this.config,
      code:this.code,
    }
  }
  return error
}
```
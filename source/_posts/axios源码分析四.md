---
title: axios源码分析四
date: 2021-07-01 23:03:37
tags:
---

### adapter

#### getDefaultAdapter

```js
function getDefaultAdapter(){
  var adapter
  // window浏览器环境下 请求
  if(typeof XMLHttpRequest !== 'undefined'){
    adapter = require('./adapters/xhr')
  }else if(typeof process !== 'undefined' && Object.prototype.toString.call(process) === '[object process]'){
    // node环境下 请求
    adapter = require('./adapters/http')
  }
  return adapter
}
```

#### xhr

```js
function xhrAdapter(config){
  return new Promise(function(resolve,reject){
    var requestData = config.data
    var requestHeaders = config.headers
    var responseType = config.responseType
    // fromData请求数据则请求数据类型由浏览器设置
    if(utils.isFormData(requestData)){
      delete requestHeaders['Content-Type']
    }
    var request = new XMLHttpRequest()
    // 用户验证
    if(config.auth){
      var username = config.auth.username || ''
      var password = config.auth.password ? unescape(encodeURIComponent(config.auth.password)) : ''
      requestHeaders.Authorization = 'Basic' + bota(username + ':' + password)
    }
    var fullPath = buildFullPath(config.baseURL,config.url)
    // 发起请求
    request.open(config.method.toUpperCase(),buldURL(fullPath,config.params,config.paramsSerializer),true)
    // 设置请求超时
    request.timeout = config.timeout
    function onloadend(){
      if(!request){
        return
      }
      // 格式化响应头
      const responseHeaders = 'getAllResponseHeaders' in request ? parseHeaders(request.getAllResponseHeaders()) : null
      // 获取响应
      var responseData = !responseType || responseType=== 'text' || responseType === 'json' ? response.responseText : response.response
      var response = {data:responseData,status:request.status,statusText:request.statusText,headers:responseHeaders,config,request}
      // 判断响应状态是否符合
      settle(resolve,reject,response)
      request = null
    }
    if('onloadend' in request){
      // 判断请求结束
      request.onloadend = onloadend
    }else{
      request.onreadystatechange = function(){
        // 请求状态未结束
        if(!request || !request.readyState !== 4){
          return
        }
        if(request.status === 0 && !(request.responseURL && request.responseURL.indexOf('file:') === 0)){
          // file://协议请求即使成功状态也是0
          return
        }
        // 请求成功处理
        setTimeout(onloadend)
      }
    }
    request.onabort = function(){
      if(!request) return
      // 请求中断/报错
      reject(createError('Request aborted',config,'ECONNABORTED',request))
      request = null
    }
    // 请求出错
    request.onerror = function(){
      reject(createError('Network Error',config,null,request))
      request = null
    }
    // 请求超时
    request.ontimeout = function(){
      var timeoutErrorMessage = 'timeout of'+config.timeout+ 'ms exceeded'
      if(config.timeoutErrorMessage){
        timeoutErrorMessage = config.timeoutErrorMessage
      }
      reject(createError(timeoutErrorMessage,config,config.transitional && config.transitional.clarifyTimeoutError ? 'ETIMEDOUT' : 'ECONNABORTED',request))
      request = null
    }
    // 设置xsrfcookie
    if(utils.isStandardBrowserEnv()){
      var xsrfValue = (config.withCredentials || isURLSameOrigin(fullPath)) && config.xsrfCookieName ? cookies.read(config.xsrfCookieName) :null
      if(xsrfValue){
        requestHeaders[config.xsrfCookieName] = xsrfValue
      }
    }
    if('setRequestHeader' in request){
      utils.forEach(requestHeaders,function(val,key){
        // 如果请求传输数据为空则删除Content-Type请求头
        if(typeof requestData === 'undefined' && key.toLowerCase() === 'Content-Type'){
          delete requestHeaders[key]
        }else{
          // 设置请求头
          request.setRequestHeader(key,val)
        }
      })
    }
    // 请求传输cookie携带
    if(!utils.isUndefined(config.withCredentials)){
      request.withCredentials = !!config.withCredentials
    }
    // 响应数据格式
    if(responseType && responseType !== 'json'){
      request.responseType = config.responseType
    }
    // 添加下载进度
    if(typeof config.onDownloadProgress === 'function'){
      request.addEventListener('progress',config.onDownloadProgress)
    }
    // 添加上传进度
    if(typeof config.onUploadProgress === 'function' && request.upload){
      request.upload.addEventListener('progress',config.onUploadProgress)
    }
    // 判断中断请求
    if(config.cancelToken){
      config.cancelToken.promise.then(function(cancel){
        if(!request){
          return
        }
        request.abort()
        reject(cancel)
        request = null
      })
    }
    if(!requestData){
      requestData = null
    }
    // 发送请求数据
    request.send(requestData)
  })
}
```

##### buildFullPath 

```js
//  获取请求路径
function buildFullPath(baseURL,requestedURL){
  if(baseURL && !isAbsoluteURL(requestedURL)){
    return combineURLs(baseURL,requestedURL)
  }
  return requestedURL
}
```

##### isAbsoluteURL

```js
// 判断是根路径包含协议
function isAbsoluteURL(url){
  return /^([a-z][a-z\d\+\-\.]*:)?\/\//i.test(url)
}
```

##### combineURLs

```js
// 合并请求路径
function combineURLs(baseUrl,relativeUrl){
  return relativeUrl ? baseUrl.replace(/\/+$/,'')+'/'+relativeUrl.replace(/^\/+/,'') : baseUrl
```

##### parseHeaders

```js
var ignoreDuplicateOf = ['age','authorization','content-length','content-type','etag','expires','from','host','if-modified-since','if-unmodified-since','last-modified','location','max-forwards','proxy-authorization','referer','retry-after','user-agent']
function parseHeaders(headers){
  var parsed = {}
  var key,val,i;
  if(!headers){return parsed}
  utils.forEach(headers.split('\n'),function(line){
    i = line.indexOf(':')
    //  获取key/val
    key = utils.trim(line.substr(0,i)).toLowerCase()
    val = utils.trim(line.substr(i+1))
    if(key){
      //ignoreDuplicateOf 只取一次
      if(parsed[key] && ignoreDuplicatedOf.indexOf(key) >= 0){
        return
      }
      // 获取header
      if(key === 'set-cookie'){
        parsed[key] = (parsed[key] ? parsed[key] : []).concat([val])
      }else{
        parsed[key] = parsed[key] ? parsed[key] +','+val:val
      }
    }
  })
  return parsed
}
```

##### settle

```js
function settle(resolve,reject,response){
  var validateStatus = response.config.validateStatus
  // 判断响应的状态码是否符合进行返回
  if(!response.status|| !validateStatus || validateStatus(response.status)){
    resolve(response)
  }else{
    reject(createError('Request Failed with status code:' + response.status,response.config,null,response.request,response))
  }
}
```

##### createError

```js
function createError(message,config,code,request,response){
  var error = new Error(message)
  return enhanceError(error,config,code,request,response)
}
```

##### cookies

```js
// 公共cookie库
module.exports = {
  utils.isStandardBrowserEnv() ? (function standardBrowserEnv(){
    return {
      // 写入cookie
      write:(name,value,expires,path,domain,secure) => {
        var cookie = []
        cookie.push(name +'='+encodeURIComponent(value))
        if(utils.isNumber(expires)){
          cookie.push('expires='+new Date(expires).toGMTString())
        }
        if(utils.isString(path)){
          cookie.push('path='+path)
        }
        if(utils.isString(domain)){
          cookie.push('domain='+domain)
        }
        if(secure == true){
          cookie.push('secure')
        }
        document.cookie = cookie.join(';')
      },
      // 读取cookie
      read:(name) => {
        var match = document.cookie.match(new RegExp(`(^|;\\s*)(${name})=([^;]*)`))
        return match ? decodeURIComponent(match[3]) : null
      },
      // 移除cookie/设置超时时间
      remove:(name)=>{
        this.write(name,'',Date.now() - 86400000)
      }
    }
  })() : (function nonStandardBrowserEnv(){
    return {
      write:() => {},
      read:() => null,
      remove:() => {}
    }
  })()
}
```

##### isStandardBrowserEnv

```js
function isStandardBrowserEnv(){
  // 检验当前环境是浏览器环境非RN环境
  if(typeof navigator !== 'undefined' && (navigator.product === 'ReactNative' || navigator.product === 'NativeScript' || navigator.product === 'NS')){
    return false
  }
  return typeof window !== 'undefined' && typeof document !== 'undefined'
}
```
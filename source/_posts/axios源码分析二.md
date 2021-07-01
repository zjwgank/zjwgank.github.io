---
title: axios源码分析二
date: 2021-06-29 22:37:03
tags:
---

### Axios

```js
function Axios(instanceConfig){
  // 传递配置
  this.defaults = instanceConfig
  // 拦截器
  this.interceptors = {
    request: new InterceptorManager(),
    respons: new InterceptorManager()
  }
}
Axios.prototype.request = function request(config){
  if(typeof config === 'string'){
    // 多个参数传递
    config = arguments[1] || {}
    config.url = arguments[0]
  }else{
     config = config || {}
  }
  config = mergeConfig(this.default,config)
  // 默认声明请求方法
  if(config.method){
    config.method = config.method.toLowerCase()
  }else if(this.defaults.method){
    config.method = this.defaults.method.toLowerCase()
  }else{
    config.method = 'get'
  }
  var transitional = config.transitional
  if(transitional !== undefined){
    validator.assertOptions(transitional,{
      slientJSONParsing: validators.transitional(validators.boolean,'1.0.0'),
      forcedJSONParsing: validators.transitional(validators.boolean,'1.0.0'),
      clarifyTimeoutError:validators.transitional(validators.boolean,'1.0.0') 
    },false)
  }
  // 存储请求拦截器
  var requestInterceptorChain = []
  // 默认同步请求拦截 即先拦截再请求
  var synchronousRequestInterceptors = true
  this.interceptors.request.forEach(function(interceptor){
    // 运行时判断请求是否继续
    if(typeof interceptor.runWhen === 'function' && interceptor.runWhen(config) === false){
      return
    }
    synchronousRequestInterceptors = synchronousRequestInterceptors && interceptor.synchronous
    // 存储拦截成功/失败
    requestInterceptorChain.unshift(interceptor.fulfilled,interceptor.rejected)
  })
  // 存储请求后的拦截
  var responseInterceptorChain = []
  this.interceptors.response.forEach(function(interceptor){
    responseInterceptorChain.push(interceptor.fulfilled,interceptor.rejected)
  })
  var promise
  // 请求拦截非同步
  if(!synchronousRequestInterceptors){
    // 请求的队列 请求拦截成功/失败/请求/响应拦截/响应拦截失败
    var chain = [dispatchRequest,undefined]
    Array.prototype.unshift.apply(chain,requestInterceptorChain)
    chain.conat(responseInterceptorChain)
    // 返回请求配置
    primise = Promise.resolve(config)
    // 依此执行请求的队列
    while(chain.length){
      promise = promise.then(chain.shift(),chain.shift())
    }
    return promise
  }
  //  同步请求+拦截
  var newConfig = config
  while(requestInterceptorChain.length){
    var onFulfilled = requestInterceptorChain.shift()
    var onRejected = requestInterceptorChain.shift()
    // 解析请求拦截配置
    try{
      newConfig = onFulfilled(config)
    }catch(error){
      onRejected(err)
      break
    }
  }
// 发起请求
  try{
    promise = dispatchRequest(newConfig)
  }catch(error){
    return Promise.reject(error)
  }
  //  请求成功后响应拦截
  while(responseInterceptorChain.length){
    promise = promise.then(responseInterceptorChain.shift(),responseInterceptorChian.shift())
  }
  return promise
}
// 获取uri
Axios.prototype.getUri = function getUri(config){
  config = mergeConfig(this.defaults,config)
  return buildURL(config.url,config.params,config.paramsSerializer).replace(/^\?/,'')
}
// 定义delete/get/head/options方法
utils.forEach(['delete','get','head','options'],function(method){
  Axios.prototype[method] = function(url,config){
    return this.request(mergeConfig(config||{},{
      method,
      url,
      data:(config || {}).data
    }))
  }
})
// 定义post/put/patch方法
utils.forEach(['post','put','patch'],function(method){
  Axios.prototype[method] = function(url,data,config){
    return this.request(mergeConfig(config || {},{
      method,url,data
    }))
  }
})

```

#### InterceptorManager

```js
// 拦截器类
function InterceptorManager(){
  // 存储拦截器操作
  this.handlers = []
}
// 存储拦截器操作
InterceptorManager.prototype.use = function use(fulfilled,rejected,options){
  this.handlers.push({
    fulfilled:fulfilled,
    rejcted:rejected,
    synchronous: options ? options.synchronous : false,
    runWhen: options ? options.runWhen : null
  })
  return this.handlers.length - 1
}
// 清除拦截器
InterceptorManager.prototype.eject = function eject(id){
  if(this.handlers[id]){
    this.handlers[id] = null
  }
}
InterceptorManager.prototype.forEach = function forEach(fn){
  utils.forEach(this.handlers,function forEachHandler(h){
    if(h !== null){
      fn(h)
    }
  })
}
```

#### mergeConfig

```js
function mergeConfig(config1,config2){
  config2 = config2 || {}
  var config = {}
  var valueFromConfig2Keys = ['url','method','data']
  var mergeDeepPropertiesKeys = ['headers','auth','proxy','params']
  var defaultToConfig2Keys = ['baseURL','transformRequest','transformResponse','paramsSerializer','timeout','timeoutMessage','withCredentials','adapter','responseType','xsrfCookieName','xsrfHeaderName','onUploadProgress','onDownloadProgress','decompress','maxContentLength','maxBodyLength','maxRedirects','transport','httpAgent','httpsAgent','cancelToken','socketPath','responseEncoding']
  var directMergeKeys = ['validateStatus']
  // target/source合并
  function getMergedValue(target,source){
    if(utils.isPlainObject(target) && utils.isPlainObject(source)){
      return utils.merge(target,source)
    }else if(utils.isPlainObject(source)){
      return utils.merge({},source)
    }else if(utils.isArray(source)){
      return source.slice()
    }
    return source
  }
  // 拓展config2到config1
  function mergeDeepProperties(prop){
    if(!utils.isUndefined(config2[prop])){
      config[prop] = getMergedValue(config1[prop],config2[prop])
    }else if(!utils.isUndefined(config1[prop])){
      config[prop] = getMergedValue(undefined,config1[prop])
    }
  }
  // 采用request配置url/method/data
  utils.forEach(valueFromConfig2Keys,function(prop){
    if(!utils.isUndefined(conifg2[prop])){
      config[prop] = getMergedValue(undefined,config2[prop])
    }
  })
  // 合并headers/auth/proxy/params
  utils.forEach(mergeDeepPropertiesKeys,mergeDeepProperties)
  // 优先选用config2配置 request
  utils.forEach(defaultToConfig2Keys.function(prop){
    if(!utils.isUndefined(config2[prop])){
      conifg[prop] = getMergedValue(undefined,config2[prop])
    }else if(!utils.isUndefined(config1[prop])){
      config[prop] = getMergedValue(undefined,config1[prop])
    }
  })
  // config1 > config2 [validateStatus]
  utils.forEach(directMergeKeys,function(prop){
    if(prop in config2){
      config[prop] = getMergedValue(config1[prop],config2[prop])
    }else if(prop in config1){
      config[prop] = getMergedValue(undefined,config1[prop])
    }
  })
  var axiosKeys = valueFromConfig2Keys.concat(mergeDeepPropertiesKeys).concat(defaultToConfig2Keys).concat(directMergeKeys)
  const otherKeys = Object.keys(config1).concat(Object.keys(config2)).filter(function filterAxiosKeys(key) {
    return axiosKeys.indexOf(key) === -1
  })
  // 合并其他属性
  utils.forEach(otherKeys,mergeDeepProperties)
  return config
}
```

#### buildURL

```js
function buildURL(url,params,paramsSerializer){
  // 参数不存在 直接返回url
  if(!params){
    return url
  }
  var serializedParams
  // 序列化参数
  if(paramsSerializer){
    serializedParams = paramsSerializer(params)
  } else if(utils.isURLSearchParams(params)){
    serializedParams = params.toString()
  }else{
    var parts = []
    //  遍历参数
    utils.forEach(params,function(val,key){
      if(val === null|| typeof val === 'undefined'){
        return 
      }
      //  判断当前的参数key是否为数组
      if(utils.isArray(val)){
        key = key + '[]'
      }else{
        val = [val]
      }
      // 获取不同格式的参数
      utils.forEach(val,function(v){
        if(utils.isDate(v)){
          v = v.toISOString()
        }else if(utils.isObject(v)){
          v = JSON.stringify(v)
        }
        parts.push(encode(key)+'='+ encode(v))
      })
    })
    // 参数合并
    serializedParams = parts.join("&")
  }
      if(serializedParams){
      var hashmarkIndex = url.indexOf('#')
      // 判断hash是否存在取非hashurl
      if(hashmarkIndex !== -1){
        url = url.slice(0,hashmarkIndex)
      }
      // 判断？是否存在组合参数
      url = (url.indexOf('?') === -1 ? '?' : '&') + serializedParams
    }
    return url
}
```

##### encode

```js
function encode(val){
  // encode之后将:/&/,/+/[/]的转义字符还原
  return encodeURIComponent(val).replace(/%3A/gi,':').relace(/%24/g,'&').replace(/%2C/gi,',').replace(/%20/g,'+').relace(/%5B/gi,'[').relace(/%5D/gi,']')
}
```

#### validator

```js
var pkg = require('../../../package.json')
var validators = {}
//  判断数据类型
['object','boolean','number','function','string','symbol'].forEach(function(type,i){
  validators[type] = function(thing){
    return typeof thing === type || 'a' + (i < 1 ? 'n' : ' ') + type
  }
})
var deprecateWarnings = {}
//  获取当前版本
var currentVerArr = pkg.version.split('.')
// 对比版本是不是更早版本
function isOlderVersion(version,thanVersion){
  var pkgVersionArr = thanVersion ? thanVersion.split('.') : currentVerArr
  var destVer = version.split('.')
  for(var i = 0; i < 3; i ++){
    if(pkgVersionArr[i] > destVer[i]){
      return true
    }else if(pkgVersionArr[i] < destVer[i]){
      reurn false
    }
  }
  return false
}
```

##### assetOptions

```js
function assetOptions(option,schema,allowUnKonwn){
  // transitional必须是个对象
  if(typeof options !== 'object'){
    throw new TypeError('options must be an object')
  }
  var keys = Object.keys(options)
  var i = keys.length
  // 遍历transitional
  while(i -- > 0){
    var opt = keys[i]
    var validator = schema[opt]
    if(validator){
      // 判断transitional
      const value = options[opt]
      // 未设置跳过/设置后进行校验
      var result = value === undefined || validator(value,opt,options)
      if(result !== true){
        throw new Error('option'+opt+'must be'+result)
      }
      continue
    }
    // 未知配置
    if(allowUnKnown !== true){
      throw new Error('unKnown option:'+opt)
    }
  }
}
```

##### transitional

```js
validators.transitional = function (validator,version,message){
  // 判断当前版本是否过期
  var isDeprecated = version && isOlderVersion(version)
  function formatMessage(opt,desc){
    return '[Axios v'+pkg.version+'] Transitional Option\''+opt+'\''+desc+(message ? '.' + message : '')
  }
  return function(value,opt,opts){
    if(validator === false){
      // JSONParsing false输出
      throw new Error(formatMessage(opt,'has been removed in'+ version))
    }
    if(isDeprecated && !deprecatedWarnings[opt]){
      deprecatedWarnings[opt] = true
      console.warn(formatMessage(opt,'has been deprecated since v '+ version + 'and will be removed in the near future'))
    }
    // 校验transitional 布尔为true
    return validator ? validator(value,opt,opts) : true
  }
}
```
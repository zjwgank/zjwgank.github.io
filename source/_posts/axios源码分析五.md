---
title: axios源码分析五
date: 2021-07-02 23:43:40
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

#### http

```js
var isHttps = /https:?/
function httpAdapter(config){
  return new Promise(function(resolve,reject){
    var resolve = function(val) {resolvePromise(val)}
    var reject = function(val){rejectPromise(val)}
    var data = config.data
    var headers = config.headers
    // 设置用户代理
    if('User-Agent' in headers || 'user-agent' in headers){
      // User-Agent不存在清除
      if(!headers['User-Agent'] && !headers['user-agent']){
        delete headers['User-Agent']
        delete headers['user-agent']
      }
    }else{
      // 设置User-Agents
      headers['User-Agent'] = 'axios/'+pkg.versiob
    }
    // 设置传输的数据
    if(data && !utils.isStream(data)){
      if(Buffer.isBuffer(data)){}else if(utils.isArrayBuffer(data)){
        data = Buffer.from(new Uint8Array(data))
      }else if(utils.isString(data)){
        data = Buffer.from(data,'utf-8')
      }else{
        return reject(createError('Data after transformation must be a string,an ArrayBuffer,a Buffer,or a Stream',config,))
      }
      headers['Content-Length'] = data.length
    }
    // 用户验证
    var auth = undefined
    if(config.auth){
      var username = config.auth.username || ''
      var password = config.auth.password || ''
      auth = username + ':' + password
    }
    //  获取请求接口
    var fullPath = buildFullPath(config.baseUrl,config.url)
    var parsed = url.parse(fullPath)
    var protocol = parsed.protocol || 'http:'
    // 用户验证信息
    if(!auth && parsed.auth){
      var urlAuth = parsed.auth.split(':')
      var urlUsername = urlAuth[0] || ''
      var urlPassword = urlAuth[1] || ''
      auth = urlUsername + ':' + urlPassword
    }
    if(auth){
      delete headers.Authorization
    }
    //  校验是否是https链接获取用户代理
    var isHttpsRequest = isHttps.test(protocol)
    var agent = isHttpsRequest ? config.httpsAgent : config.httpAgent
    // 请求配置
    var options = {
      path: buildURL(parsed.path,config.params,config.paramsSerializer).replace(/^\?/,''),
      method: config.method.toUpperCase(),
      headers,
      agent,
      agents:{http:config.httpAgent,https:config.httpsAgent},
      auth
    }
    // 设置请求地址
    if(config.socketPath){
      options.socketPath = config.socketPath
    }else{
      options.hostname = parsed.hostname
      options.port = parsed.port
    }
    var proxy = config.proxy
    if(!proxy && proxy !== false){
      var proxyEnv = protocol.slice(0,-1) + '_proxy'
      var proxyUrl = process.env[proxyEnv] || process.env[proxyEnv.toUpperCase()]
      if(proxyUrl){
        var parsedProxyUrl = url.parse(proxyUrl)
        var noProxyEnv = process.env.no_proxy|| process.env.NO_PROXY
        var shouldProxy = true
        // 判断没有配置默认代理或者当前接口与默认代理不匹配
        if(noProxyEnv){
          var noProxy = noProxyEnv.split(',').map((s) => s.trim())
          shouldProxy = !noProxy.some((proxyElement) =>{
            if(!proxyElement){
              return false
            }
            if(proxyElement=== '*'){
              return true
            }
            if(proxyElement[0] === '.' && parsed.hostname.substr(parsed.hostname.length - proxyElement)){
              return true
            }
            return parsed.hostname === proxyElement
          })
        }
        if(shouldProxy){
          // 根据配置设置接口代理
          proxy = {
            host:parsedProxyUrl.hostname,
            prot:parsedProxyUrl.port,
            protocol:parsedProxyUrl.protocol
          }
          if(parsedProxyUrl.auth){
            var proxyUrlAuth = parsedProxyUrl.auth.split(':')
            proxy.auth = {
              username:proxyUrlAuth[0],
              password:proxyUrlAuth[1]
            }
          }
        }
      }
    }
    // 设置代理
    if(proxy){
      options.headers.host = parsed.hostname + (parsed.port ? ':' + parsed.port : '')
      setProxy(options,proxy,protocol+'//'+parsed.hostname+(parsed.port ? ':'+parsed.port : '')+options.path)
    }
    var transport
    // 确定请求方法
    var isHttpsProxy = isHttpsRequest && (proxy ? isHttps.test(proxy.protocol) : true)
    if(config.transport){
      transport = config.transport
    }else if(config.maxRedirects === 0){
      transport = isHttpsProxy ? https : http
    }else{
      if(config.maxdirects){
        options.maxdirects = config.maxdirects
      }
      transport = isHttpsProxy ? httpsFollow : httpFollow
    }
    if(config.maxBodyLength > -1){
      options.maxBodyLength = config.maxBodyLength
    }
    // 创建请求
    var req = transport.request(options,(res) => {
      if(req.aborted) return
      var stream = res
      var lastRequest = res.req || req
      // 如果非连接请求，同时返回压缩内容，则进行解压
      if(res.statusCode !== 204 && lastRequest.method !== 'HEAD' && config.decompress !== false){
        switch(res.headers['content-encoding']){
          case 'gzip':
          case 'compress':
          case 'deflate':
           stream = stream.pipe(zlib.createUnzip())
           delete res.headers['content-encoding']
           break
        }
      }
      var response = {
        status:res.statusCode,
        statusText: res.statusMessage,
        headers:res.headers,
        config,
        request:lastRequest
      }
      // 针对流数据直接返回
      if(config.responseType === 'stream'){
        response.data = stream
        settle(resolve,reject,response)
      }else{
        // 其他类型数据进行读取
        var responseBuffer = []
        var totalResponseBytes = 0
        // 流数据处理
        stream.on('data',(chunk) => {
          responseBuffer.push(chunk)
          totalResponseBytes += chunk.length
          if(config.maxConentLength > -1 && totalResponseBytes > config.maxContentLength){
            stream.destroy()
            reject(createError('maxContentLength size of ' + config.maxContentLength + 'exceeded',config,null,lastRequest))
          }
        })
        stream.on('error',(err) => {
          if(req.aborted) return
          reject(enhanceError(err,config,null,lastRequest))
        })
        stream.on('end',() => {
          var responseData = Buffer.concat(responseBuffer)
          if(config.responseType !== 'arraybuffer'){
            responseData = responseData.toString(cofig.responseEncoding)
            if(!config.responseEncoding || config.responseEncoding === 'utf8'){
              responseData = utils.stripBOM(responseData)
            }
          }
          response.data = responseData
          settle(resolve,reject,response)
        })
      }
    })
    // 请求出错
    req.on('error',(err) => {
      if(req.aborted && err.code !== 'ERR_FR_TOO_MANY_REDIRECTS') return
      reject(enhanceError(err,config,null,req))
    })
    if(config.timeout){
      var timeout = parseInt(config.timeout,10)
      if(isNAN(timeout)){
        reject(createError('error trying to parse `config.timeout` to int',config,'ERR_PARSE_TIMEOUT',req))
        return
      }
      // 请求超时
      req.setTimeout(timeout,() => {
        req.abort()
        reject(createError('timeout of '+timeout+'ms exceded',config,config.transitional && config.transitional.clarifyTimeoutError ? 'ETIMEDOUT': 'ECONNABORTED',req))
      })
    }
    if(config.cancelToken){
      config.cancelToken.promise.then((cancel) => {
        if(req.aborted) return
        req.abort()
        reject(cancel)
      })
    }
    // 发起请求
    if(utils.isStream(data)){
      data.on('error',(err) => {
        reject(enhanceError(err,config,null,req))
      }).pipe(req)
    }else{
      req.end(data)
    }
  })
}
```

#### setProxy

```js
function setProxy(options,proxy,location){
  // 设置代理
  options.hostname = proxy.host
  options.host = proxy.host
  options.port = proxy.port
  options.path = proxy.location
  if(proxy.auth){
    var base64 = Buffer.from(proxy.auth.username + ':'+proxy.auth.password,'utf-8').toString('base64')
    options.headers['Proxy-Authorization'] = 'Basic' + base64
  }
  // 重定向代理设置
  options.beforeRedirect = (redirection) => {
    redirection.headers.host = redirection.host
    setProxy(redirection,proxy,redirection.href)
  }
}
```
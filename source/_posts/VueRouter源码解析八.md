---
title: VueRouter源码解析八
date: 2021-06-25 21:31:48
tags:
---

### fillParams

```js
const regexCompileCache = Object.create(null)
function fillParams(path,params,routeMsg){
  params = params || {}
  try{
    // 根据路径创建编译对象
    const filler = regexCompileCache[path] || (regexCompileCache[path] = Regexp.compile(path))
    // 如果匹配的是一个字符串则将路径匹配作为首位进行编译
    if(typeof params.pathMatch === 'string') params[0] = params.pathMatch
    return filler(params,{pretty:true})
  }catch(e){
    if(process.env.NODE_ENV !== 'production'){
      warn(typeof params.pathMatch==='string',`missing param for ${routeMsg}:${e.message}`)
    }
    return ''
  }finally{
    delete params[0]
  }
}
```

### path 

#### resolvePath

```js
function resolvePath(relative,base,append){
  // 查询路径的第一个字符
  const firstChar = relative.charAt(0)
  // 以/开头直接返回
  if(firstChar=== '/'){
    return relative
  }
  // 以search或hash开头则添加base返回
  if(firstChar === '?' || firstChar === '#'){
    return base + relative
  }
  // 存储路径每一级
  const stack = base.split('/')
  // 如果不需要追加路径或者基础路径最后一个为空,则路径移除最后一个
  if(!append || !stack[stack.length - 1]){
    stack.pop()
  }
  // 移除首/切割
  const segments = relative.replace(/^\//,'').split('/')
  for(let i = 0; i < segments.length; i ++){
    const segment = segments[i]
    // /.. 表示跳转上一个目录
    if(segment === '..'){
      stack.pop()
    }else if(segment !== '.'){
      // 非.则存储路径
      stack.push(segment)
    }
  }
  // 保证以/开头
  if(stack[0] !== ''){
    stack.unshift('')
  }
  return stack.join('/')
}
```
#### parsePath

```js
// 切割path => query/hash/path
function parsePath(path){
  let hash = ''
  let query = ''
  const hashIndex = path.indexOf('#')
  if(hashIndex >= 0){
    hash = path.slice(hashIndex)
    path = path.slice(0,hashIndex)
  }
  const queryIndex = path.indexOf('?')
  if(queryIndex >= 0){
    query = path.slice(queryIndex + 1)
    path = path.slice(0,queryIndex)
  }
  return {
    path,
    query,
    hash
  }
}
```

#### cleanPath

```js
// 替换 // => /
function cleanPath(path){
  return path.replace(/\/\//g,'/')
}
```

### query

#### parseQuery

```js
// 解析query的kv
function parseQuery(query){
  const res = {}
  // 移除处于首部的？/#/& 
  query = query.trim().replace(/^(\?|#|&)/,'')
  if(!query){
    return res
  }
  query.split('&').forEach(param => {
    const parts = param.replace(/\+/g,'').split('=')
    const key = decode(parts.shift())
    const val = parts.length > 0 ? decode(parts.join('=')): null
    if(res[key] === undefied){
      res[key] = val
    }else if (Array.isArray(res[key])){
      res[key].push(val)
    }else{
      res[key] = [res[key],val]
    }
  })
  return res
}
```

#### resolveQuery

```js
function resolveQuery(query,extraQuery,_parseQuery){
  // 判断采用自定义或者默认的query解析
  const parse = _parseQuery || parseQuery
  let parsedQuery
  // 解析query
  try{
    parsedQuery = parse(query || '')
  }catch(e){
    process.env.NODE_ENV !== 'production' && warn(false,e.message)
    parsedQuery = {}
  }
  // 针对query进行扩展
  for(const key in extraQuery){
    const value = extraQuery[key]
    parsedQuery = Array.isArry(value) ? value.map(castQueryParamValue) : castQueryParamValue
  }
  return parsedQuery
}
// 返回对象或者null或者String对象
const castQueryParamValue = value => (value== null || typeof value === 'object' ? value : String(value) )
```

#### stringifyQuery

```js
// 将query的kv合并成字符串
function stringifyQuery(obj){
  const res = obj ? Object.keys(obj).map(key => {
    const val = obj[key]
    if(val === undefined){
      return ''
    }
    if(val === null){
      return encode(key)
    }
    if(Array.isArray(val)){
      const result = []
      val.forEach(val2 => {
        if(val2 === undefined){
          return
        }
        if(val2 === null){
          result.push(encode(key))
        }else{
          result.push(encode(key) + '=' + encode(val2))
        }
      })
      return result.join('&')
    }
    return encode(key) + '=' + encode(val)
  }).filter(x => x.length > 0).join('&') : null
}
```

### resolve-components

#### flatten

```js
const flatten = (arr) => Array.prototype.concat.apply([],arr)
```

#### flatMapComponents

```js
const flatMapComponents = (matched,fn) => flatten(matched.map(m => Object.keys(m.components).map(key => fn(m.components[key],m.instances[key],m,key))))
```

#### isESModule

```js
// 判断ES6 Module
const hasSymbol = typeof Symbol === 'function' && typeof Symbol.toStringTag === 'symbol'
const isESModule = (obj) => obj.__esModule || (hasSymbol && obj[Symbol.toStringTag] === 'Module')
```

#### resolveAsyncComponents

```js
function resolveAsyncComponents(matched){
  return (to,from,next) => {
    let hasASync = false
    let pending = 0
    let error = null
    // def组件/_实例/match路由对象/key组件key
    flatMapComponents(matched,(def,_,match,key) => {
      // 如果是个函数组件且cid未添加视为异步组件
      if(typeof def === 'function' && def.cid === undefined){
        hasAsync = true
        pending ++
        const resolve = once(resolvedDef => {
          // 如果是ES6模块则通过default获取组件
          if(isESModule(resolvedDef)){
            resolvedDef = resolvedDef.default
          }
          // 解析组件
          def.resolved = typeof resolvedDef === 'function' ? resolvedDef : _Vue.extend(resolvedDef)
          match.components[key] = resolvedDef
          // 如果组件解析完成，执行下一步
          pending --
          if(pending <= 0){
            next()
          }
        })
        const reject = once(reason => {
          const msg = `Failed to resolve async component ${key}:${reason}`
          process.env.NODE_ENV !== 'production' && warn(false,msg)
          if(!error){
            error = isError(reason) ? reason : new Error(msg)
            next(error)
          }
        })
        let res
        // 解析组件
        try{
          res = def(resolve,reject)
        }catch(e){
          reject(e)
        }
        if(typeof res.then === 'function'){
          res.then(resolve,reject)
        }else{
          const comp = res.component
          if(comp && typeof comp.then === 'function'){
            comp.then(resolve,reject)
          }
        }
      }
    })
    if(!hasAsync) next()
  }
}
```

### route

```js
// 匹配 /？结尾
const trailingSlashRE = /\/?$/
```

#### createRoute

```js
function createRoute(record,location,redirectedFrom,router){
  // 创建route对象
  const stringifyQuery = router && router.options.stringifyQuery
  let query = location.query || {}
  try{
    query = clone(query)
  }catch(e){}
  const route = {
    name: location.name || (record && record.name),
    meta: (record && record.meta) || {},
    path: location.path || '/',
    hash: location.hash || '',
    query,
    params: location.params || {},
    fullPath: getFullPath(location,stringifyQuery),
    matched: record ? formatMatch(record) : {}
  }
  if(redirectedFrom){
    route.redirectedFrom = getFullPath(redirectedFrom,stringifyQuery)
  }
  return Object.freeze(route)
}
```

#### clone 

```js
// 深拷贝
function clone(value){
  if(Array.isArray(value)){
    return value.map(clone)
  }else if(value && typeof value === 'object'){
    const res = {}
    for(const key in value){
      res[key] = clone(value[key])
    }
    return res
  }else{
    return value
  }
}
```

#### START

```js
const START = createRoute(null,{path:'/'})
```

#### formatMatch

```js
// 递归寻找当前路由的父级作为匹配对象
function formatMatch(record){
  const res = []
  while(record){
    res.unshift(record)
    record = record.parent
  }
  return res
}
```

#### getFullPath

```js
// 根据配置的或者默认的query解析恢复当前路径
function getFullPath({path,query = {},hash = ''},_stringifyQuery){
  const stringify = _stringifyQuery || stringifyQuery
  return (path || '/') + stringify(query) + hash
}
```

#### isSameRoute

```js
// 判断两个路由对象是否相等
function isSameRoute(a,b,onlyPath){
  if(b === START){
    return a === b
  }else if(!b){
    return false
  }else if(a.path && b.path){
    return a.path.replace(trailingSlashRE,'') === b.path.replace(trailingSlashRE,'') && (onlyPath || (a.hash === b.hash && isObjectEqual(a.query,b.equery)))
  }else if(a.name && b.name){
    return a.name === b.name && (onlyPath || (a.hash === b.hash && isObjectEqual(a.query,b.equery) && isObjectEqual(a.params,b.params)))
  }else{
    return false
  }
}
```

#### isObjectEqual

```js
// 比对两个对象是否相等
function isObjectEqual(a={},b={}){
  if(!a || !b) return a === b
  const aKeys = Object.keys(a).sort()
  const bKeys = Object.keys(b).sort()
  if(aKeys.length !== bKeys.length){
    return false
  }
  return aKeys.every((key,i) => {
    const aVal = a[key]
    const bKey = bKeys[i]
    if(bKey !== key) return false
    const bVal = b[bKey]
    if(aVal === null || bVal === null) return aVal === bVal
    if(tyoeof aVal === 'object' && typeof bVal === 'object'){
      return isObjectEqual(aVal,bVal)
    }
    return String(aVal) === String(bVal)
  })
}
```

#### isIncludedRoute

```js
//  判断路由是否包含
function isIncludedRoute(current,target){
  return (current.path.replace(trailingSlashRE,'').indexOf(target.path.replace(trailingSlashRE,'')) === 0 && (!target.hash || current.hash === target.hash) && queryIncludes(current.query,target.query))
}

function queryIncludes(current,target){
  for(const key in target){
    if(!(key in current)){
      return false
    }
  }
  return true
}
```

#### handleRouteEntered

```js
function handleRouteEntered(route){
  //   依此获取当前路由的父级
  for(let i = 0; i < route.matched.length; i ++){
    const record = route.matched[i]
    for(const name in record.instances){
      // 获取匹配到路由的vue实例和enterCbs回调
      const instance = record.instanced[name]
      const cbs = record.enterCbs[name]
      if(!instance || !cbs) continue
      // 此时默认都会执行清除vue实例对应的回调
      delete record.enterCbs[name]
      // 依此执行enter回调
      for(let i = 0; i < cbs.length; i++){
        if(!instance._isBeingDestroyed) cbs[i](instance)
      }
    }
  }
}
```
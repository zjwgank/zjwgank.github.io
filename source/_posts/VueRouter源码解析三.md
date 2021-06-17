---
title: VueRouter源码解析三
date: 2021-06-09 22:16:08
tags:
---

```js
class VueRouter{
  ...
  constructor(options){
    // 挂载的vue实例
    this.app = null
    this.apps = []
    this.options = options
    // 路由守卫方法
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    // 创建路由匹配对象
    this.matcher = createMatcher(options.routes || [],this)
  ...
}
```
### createMatcher

```js
export function createMatcher(routes,router){
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
  // 动态更新路由对象
  function addRoutes(routes){
    createRouteMap(routes,pathList,pathMap,nameMap)
  }
  function addRoute(parentOrRoute,route){
    // 判断路由或者父节点是否存在
    const parent = ( typeof parentOrRoute !== 'object' ) ? nameMap[parentOrRoute] : undefined
    // 根据要添加的路由更新路由对象
    createRouteMap([route || parentOrRoute],pathList,pathMap,nameMap,parent)
    // 向父路由下更新路由对象
    if(parent){
      createRouteMap(parent.alias.map(alias => ({path:alias,children:[route]})),pathList,pathMap,nameMap,parent)
    }
  }
  // 根据路由路径获取路由对象
  function getRoutes(){
    return pathList.map(path => pathMap[path])
  }
  function match(raw,currentRoute,redirectedFrom){
    // 格式化跳转的location
    const location = normalizeLocation(raw,currentRoute,false,router)
    const { name } = location
    if(name){
      const record = nameMap[name]
      if(process.env.NODE_ENV !== 'production'){
        ...// 路由对象必须存在
      }
      // 如果路由对象不存在，创建路由
      if(!record) return _createRoute(null,location)
      // 获取路由传输参数
      const paramNames = record.regex.keys.filter(key => !key.optional).map(key => key.name)
      // 设置参数
      if(typeof location.params !== 'object'){
        location.params = {}
      }
      // 传输路由参数到跳转参数
      if(currentRoute && typeof currentRoute.params === 'object'){
        for(const key in currentRoute.params){
          if(!(key in loaction.params) && paramNames.indexOf(key) > -1 ){
            location.params[key] = currentRoute.params[key]
          }
        }
      }
      location.path = fillParams(record.path,location.params,`name route "${name}"`)
      // 创建路由
      return _createRoute(record,location,redirectedFrom)
    }else if(location.path){
      location.params = {}
      // 遍历路由列表
      for(let i = 0; i < pathList.length; i ++){
        const path = pathList[i]
        const record = pathMap[path]
        if(matchRoute(record.regex,location.path,location.params)){
          return _createRoute(record,location,redirectedFrom)
        }
      }
    }
    // 创建location路由
    return _createRoute(null,location)
  }

  function redirect(record,location){
    const originalRedirect = record.redirect
    // 重定向路由
    let redirect = typeof originalRedirect === 'function' ? originalRedirect(createRoute(record,location,null,router)) : originalRedirect
    // 设置重定向路由
    if(typeof redirect === 'string'){
      redirect = {path:redirect}
    }
    if(!redirect || typeof redirect !== 'object'){
      ...// redirect必须
      // 返回当前路由对象
      return _createRoute(null,location)
    }
    // 获取query/params/hash
    const re = redirect
    const { name, path } = re
    let { query, params, hash} = location
    query = re.hasOwnProperty('query') ? re.query : query
    params = re.hasOwnProperty('params') ? re.params : params
    hash = re.hasOwnProperty('hash') ? re.hash : hash
    if(name){
      const targetRecord = nameMap[name]
      ...//  路由对象必须
      // 返回匹配到的路由对象
      return match({_normalized:true,name,query,hash,params},undefined,location)
    }else if(path){
      // 解析路由路径
      const rawPath = resolveRecordPath(path,record)
      const resolvedPath = fillParams(rawPath,params,`redirect route with path "${rawPath}"`)
      return match({_normalized:true,path:resolvedPath,query,hash},undefined,location)
    }else{
      ...//  path/name必须
      // 创建当前路由对象
      return _createRoute(null,location)
    }
  }

  function alias(record,location,matchAs){
    const aliasedPath = fillParams(matchAs,location.params,`aliased route with path "${matchAs}"`)
    // 返回匹配的路由对象或者当前路由对象
    const aliasedMatch = match({_normalized:true,path:aliasedPath})
    if(aliasedMatch){
      // 根据匹配到的路由创建路由对象
      const matched = aliasedMatch.matched
      const aliasedRecord = matched[matched.length - 1]
      location.params = aliasedMath.params
      return _createRoute(aliasedRecord,location)
    }
    // 返回当前路由对象
    return _createRoute(null,location)
  }

  function _createRoute(record,location,redirectedFrom){
    // 重定向路由对象或者返回当前路由
    if(record && record.redirect){
      return redirect(record,redirectedFrom || location)
    }
    //  根据匹配到的路由
    if(record && record.matchAs){
      return alias(record,location,record.matchAs)
    }
    // 创建路由对象
    return createRoute(record,location,redirectedForm,router)
  }
  return {
    match,
    addRoute,
    getRoutes,
    addRoutes
  }
}
```
#### createRouteMap

```js
export function createRouteMap(routes,oldPathList,oldPathMap,oldNameMap,parentRoute){
  // 初始化pathList/pathMap/nameMap
  const pathList = oldPathList || []
  const pathMap = oldPathMap || Object.create(null)
  const nameMap = oldNameMap || Object.create(null)
  // 根据路由配置生成路由对象/路由
  routes.forEach(route => {
    addRouteRecord(pathList,pathMap,nameMap,route,parentRoute)
  })
  // 保证*的路由匹配在路由列表的最后
  for(let i = 0, l = pathList.length; i < l; i ++){
    if(pathList[i] === '*'){
      pathList.push(pathList.slice(i,1)[0])
      l --
      i --
    }
  }
  if(process.env.NODE_ENV !== 'production'){
    ...// 路由路径不是以/或者*开头警告
  }
  return {
    pathList, // 路由路径
    pathMap, //  根据路径存储路由对象
    nameMap // 根据路由名称存储路由对象
  }
}
```

#### addRouteRecord

```js
function addRouteRecord(pathList,pathMap,nameMap,route,parent,matchAs){
  const { path, name } = route
  if(process.env.NODE_ENV !== 'production'){
    ...// path必需/component不能为字符串/path中不能包含未转义的字符
  }
  // 编译正则选项
  const pathToRegexpOptions = route.pathToRegexpOptions || {}
  //  格式化当前节点路径
  const normalizedPath = normalizePath(path,parent,pathToRegexpOptions.strict)
  // 路由匹配规则是否大小写敏感
  if(typeof route.caseSensitive === 'boolean'){
    pathToRegexpOptions.caseSensitive = route.caseSensitive
  }
  const record = {
    path: normalizedPath, // 格式化路由路径
    regex: compileRouteRegex(normalizedPath,pathToRegexpOptions), // 创建路径匹配正则表达式
    components:route.components || { default:route.component}, // 当前路径的渲染组件
    alias: route.alias ? typeof route.alias === 'string' ? [route.alias] : route.alias : [], // 路由链接
    instances:{},
    enteredCbs:{}, // 路由守卫
    name, // 路由名称
    parent, // 父节点
    matchAs,
    redirect:route.redirect, // 重定向
    beforeEnter:router.beforeEnter, // 路由守卫
    meta:route.meta || {},
    props: route.props === null ? {} : route.components ? route.props : {default:route.props}
  }
  // 路由存在子节点
  if(route.children){
    if(process.env.NODE_ENV !== 'production'){
      ...// 子节点必须拥有redirect
    }
    // 扩展子节点路径到pathList/pathMap
    route.children.forEach(child => {
      const childMatchAs = matchAs ? cleanPath(`${matchAs}/${child.path}`) : undefined
      addRouteRecord(pathList,pathMap,nameMap,child,record,childMathAs)
    })
  }
  // 存储路由路径和路由对象
  if(!pathMap(record.path)){
    pathList.push(record.path)
    pathMap[record.path] = record
  }
  if(route.alias !== undefined){
    const aliases = Array.isArray(route.alias) ? route.alias : [route.alias]
    for(let i = 0; i < aliases.length; i ++){
      const alias = aliases[i]
      if(process.env.NODE_ENV !== 'production' && alias === path){
        ... // 移除alias == path的路由对象
        continue
      }
      const aliasRoute = {path: alias, children: route.children }
      // 根据alias扩展pathMap/pathList
      addRouteRecord(pathList,pathMap,nameMap,aliasRoute,parent,record.path || '/')
    }
  }
  if(name){
    if(!nameMap[name]){
      nameMap[name] = record
    }else if(process.env.NODE_ENV !== 'production' && !matchAs){
      ... // 重复生成路由对象
    }
  }
}
```

#### normalizePath

```js
function normalizePath(path,parent,strict){
  // 非严格匹配，最后一个字符串为/时替换为空
  if(!strict) path = path.replace(/\/$/,'')
  // 判断路径正常
  if(path[0] === '/') return path
  // 判断为根路径
  if(parent == null) return path
  // 格式化路径，替换//为/
  return cleanPath(`${parent.path}/${path}`)
}
```

#### normalizeLocation

```js
function normalizeLocation(raw,current,append,router){
  // 解析跳转路径
  let next = typeof raw === 'string' ? { path: raw } : raw
  // 判断location是否处理过
  if(next._normalized){
    return next
  }else if(next.name){
    // 存在路由名称添加跳转参数返回
    next = extend({},raw)
    const params = next.params
    if(params && typeof params === 'object'){
      next.params = extend({},params)
    }
    return next
  }
  // 匹配不到路由名称和路径
  if(!next.path && next.params && current){
    next = extend({},next)
    next._normalized = true
    const params = extend(extend({},current.params),next.params)
    // 当前路由name存在不跳转，跳转参数合并
    if(current.name){
      next.name = current.name
      next.params = params
    }else if(current.matched.length){
      const rawPath = current.matched[current.matched.length - 1].path
      // 设置跳转路径为当前 路由最后的一个匹配
      next.path = fillParams(rawPath,params,`path ${current.path}`)
    }else if(process.env.NODE_ENV !== 'production'){
      warn(false,`relative params navigation requires a current route`)
    }
    return next
  }
  // 解析跳转路径
  const parsedPath = parsePath(next.path || '')
  // 获取当前路径
  const basePath = (current && current.path) || '/'
  // 解析路径
  const path = parsedPath.path ? resolvePath(parsedPath.path,basePath,append||next.append) : basePath
  // 解析query
  const query = resolveQuery(parsedPath.query,next.query,router && router.options.parseQuery)
  // hash
  let hash = next.hash || parsedPath.hash
  if(hash && hash.charAt(0) !== '#'){
    hash = `#${hash}`
  }
  return {
    _normalized:true,
    path,
    query,
    hash
  }

}
```
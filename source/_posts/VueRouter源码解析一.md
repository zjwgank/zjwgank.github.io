---
title: VueRouter源码解析一
date: 2021-05-30 22:55:37
tags:
---

### 入口

#### package.json

```js
{
  ...
  // 模块
  module:'dist/vue-router.esm.js'
  ...
  scripts:{
    ...
    // build
    build:'node build/build.js'
  }
}
```

#### build.js

```js
// 模块配置
const configs = require('./configs')
// 根据配置生成文件
build(configs)
function build(builds){
  let built = 0
  const total = builds.length
  // 递增执行rollup build
  const next = () => {
    buildEntry(builds[built]).then(() => {
      built ++
      if(built < total){
        next()
      }
    }).catch(logError)
  }
  next()
}
```
#### config.js

```js
module.exports = [
  ...
  {file: resolve('dist/vue-router.esm.js'), format: 'es'}
  ...
].map(getConfig)
//  生成rollup配置
function getConfig(opts){
  const config = {
    input:{
      // 入口文件
      input: resolve('src/index.js')
      ...
    }
    ...
  }
}
```
### VueRouter

```js
export default class VueRouter{
  static install
  static version
  static isNavigationFailure
  static NavigationFailureType
  static START_LOCATION
  app,apps,ready,readyCbs,options,mode,history,matcher,fallback,beforeHooks,resolveHooks,afterHooks
  constuctor(options = {}){
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    // 路由方法
    this.matcher = createMatcher(options.routes || [],this)
    // 默认hash模式
    let mode = options.mode || 'hash'
    // 当浏览器下history模式不支持pushState采用hash模式
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if(this.fallback){
      mode = 'hash'
    }
    // 非浏览器环境采用abstract模式
    if(!inBrowser){
      mode = 'abstract'
    }
    this.mode = mode
    // 根据mode采用不同的路由
    switch(mode){
      case 'history':
      this.history = new HTML5History(this,options.base)
      break
      case 'hash':
      this.history = new HashHistory(this,options.base)
      break;
      case 'abstract':
      this.history = new AbstractHistory(this,options.base)
      break
      default:
      if(process.env.NODE_ENV !== 'production'){
        assert(false,`invalid mode ${mode}`)
      }
    }
  }

  match(raw,current,redirectedFrom){
    // 匹配路由
    return this.matcher.match(raw,current,redirectedFrom)
  }
  get currentRoute(){
    // 获取当前路由
    return this.histroy && this.history.current
  }
  init(app){
    // VueRouter使用应该在Vue.use(VueRouter)后
    process.env.NODE_ENV !== 'production' && assert(install.installed,`not installed. Make sure to call Vue.use(VueRouter) before creating root instance`)
    // 传入vue实例
    this.apps.push(app)
    // vue执行销毁
    app.$once('hook:destroyed',() => {
      // 判断vue实例存在,销毁实例,销毁history
      const index = this.apps.indexOf(app)
      if(index > -1) this.apps.splice(index,1)
      if(this.app === app) this.app = this.apps[0] || null
      if(!this.app) this.history.teardown()
    })
    // 如果此时实例存在,直接采用
    if(this.app){
      return
    }
    this.app = app
    const history = this.history
    if(history instanceof HTML5History || history instanceof HashHistory){
      //  history/hash模式下切换路由滚动到相应的位置
      const handleInitialScroll = routeOrError => {
        const from = history.current
        const expectScroll = this.options.scrollBehavior
        const supportScroll = supportPushState && expectScroll
        if(supportScroll && 'fullPath' in routeOrError){
          handleScroll(this,routeOrError,from,false)
        }
      }
      const setupListeners = routeOrError => {
        history.setupListeners()
        handleInitialScroll(routeOrError)
      }
      // 路由切换,页面滚动
      history.transitionTo(history.getCurrentLocation(),setupListeners,setupListeners)
    }
    // 添加路由更新回调
    history.listen(route => {
      this.apps.forEach(app => {
        app._route = route
      })
    })
  }
  beforeEach(fn){
    // 注册访问前的方法,同时调用后清除
    return registerHook(this.beforeHooks,fn)
  }
  beforeResolve(fn){
    // 注册解析前的方法,同时调用后清除
    return registerHook(this.resolveHooks,fn)
  }
  afterEach(fn){
    // 注册访问后的方法,同时调用后清除
    return registerHook(this.afeterHooks,fn)
  }
  onReady(cb,errorCb){
    // 注册路由初始化完成的回调
    this.history.onReady(cb,errorCb)
  }
  onError(errorCb){
    // 注册路由失败的回调方法
    this.history.onError(errorCb)
  }
  push(location,onComplete,onAbort){
    // 路由切换,触发路由切换回调
    if(!onComplete && !onAbort && typeof Promise !== 'undefied'){
      return new Promise((resolve,reject) =>{
        this.history.push(location,resolve,reject)
      })
    }else{
      this.history.push(location,onComplete,onAbort)
    }
  }
  replace(location,onComplete,onAbort){
    // 路由切换,触发路由切换后的回调
    if(!onComplete && !onAbort && typeof Promise !== 'undefied'){
      return new Promise((resolve,reject) => {
        this.history.replace(location,resolve,reject)
      })
    }else{
      this.history.replace(location,onComplete,onAbort)
    }
  }
  // 路由切换
  go(n){
    this.history.go(n)
  }
  back(){
    this.go(-1)
  }
  forward(){
    this.go(1)
  }
  getMatchedComponent(to){
        // 解析路由或采用当前路由
    const route = to ? to.matched ? to : this.resolve(to).route : this.currentRoute
    if(!route){
      return []
    }
    // 获取匹配到的组件
    return [].concat.apply([],route.matched.map(m =>  Object.keys(m.components).map(key => m.components[key])))
  }
  resolve(to,current,append){
    current = current || this.history.current
    // 解析路由
    const location = normalizeLocation(to,current,append,this)
    // 获取路由对象
    const route = this.match(location,current)
    const fullPath = route.redirectedFrom || route.fullPath
    const base = this.history.base
    // 解析连接
    const href = createHref(base,fullPath,this.mode)
    return {
      location,
      route,
      href,
      normalizedTo:location,
      resolved:route
    }
  }
  getRoutes(){
    // 获取路由模块
    return this.matcher.getRoutes()
  }
  addRoute(parentOrRoute,route){
    // 动态添加路由规则
    this.matcher.addRoute(parentOrRoute,route)
    // 如果当前的路由不是/,执行路由跳转
    if(this.histroy.current !== START){
      this.history.transitonTo(this.history.getCurrentLocation())
    }
  }
  addRoutes(parentOrRoute,route){
    this.matcher.addRoutes(parentOrRoute,route)
    if(this.history.current !== START){
      this.history.transitionTo(this.history.getCurrentLocation())
    }
  }
}
...
if(inBrowser && window.Vue){
  // 判断浏览器环境Vue存在, 引用VueRouter
  window.Vue.use(VueRouter)
}
```
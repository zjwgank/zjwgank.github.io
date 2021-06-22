---
title: VueRouter源码解析四
date: 2021-06-17 23:40:14
tags:
---

### history

```js
    // 路由模式默认为hash
    let mode = options.mode || 'hash'
    // 当路由history时但是不支持pushState则采用hash模式
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if(this.fallback){
      mode = 'hash'
    }
    // 非浏览器采用abstract模式
    if(!inBrowser){
      mode = 'abstract'
    }
    this.mode = mode
    //  根据不同的路由模式，创建不同的路由操作对象
    switch(mode){
      case 'history':
        this.history = new HTML5History(this,options.base)
        break
      case 'hash':
        this.history = new HashHistory(this,options.base,this.fallback)
        break;
      case 'abstract':
        this.histroy = new AbstractHistory(this,options.base)
        break
        // 非法模式
      default:
        if(process.env.NODE_ENV !== 'production'){
          assert(false,`invalid mode: ${mode}`)
        }
    }
```

### historyBase

```js
class History {
  router,base,current,pending,cb,ready,readyCbs,readyErrorCbs,errorCbs,listeners,cleanupListeners,go,push,replace,ensureURL,getCurrentLocation,setupListeners;
  constructor(router,base){
    // 获取当前路由实例
    this.router = router
    // 格式化base
    this.base = normalizeBase(base)
    // 创建路径为/的路由对象
    this.current = START
    // 初始化状态/方法
    this.pending = null
    this.ready = false
    this.readyCbs = []
    this.readyErrorCbs = []
    this.errorCbs = []
    this.listeners = []
  }
  listen(cb){
    this.cb = cb
  }
  onReady(cb,errorCb){
    // 根据当前状态判断是否执行或者存储
    if(this.ready){
      cb()
    }else{
      this.readyCbs.push(cb)
      if(errorCb){
        this.readyErrorCbs.push(errorCb)
      }
    }
  }
  onError(errorCb){
    this.errorCbs.push(errorCb)
  }
  // 路由变换
  transitionTo(location,onComplete,onAbort){
    let route
    // 匹配要跳转的路由
    try{
      route = this.router.match(location,this.current)
    }catch(e){
      // 匹配出错，依此执行errorCb
      this.errorCbs.forEach((cb) =>{
        cb(e)
      })
      throw e
    }
    const prev = this.current
    this.confirmTransition(route,() => {
      // 跳转完成，更新当前路由，执行回调
      this.updateRoute(route)
      // 执行跳转成功回调
      onComplete && onComplete(route)
      this.ensureURL()
      // 执行跳转完成后的回调
      this.router.afterHooks.forEach(hook => hook && hook(route,prev))
    },err =>{
      // 路由切换出错
      if(onAbort){
        onAbort(err)
      }
      if(err && !this.ready){
        // 如果非重定向报错或者非根节点跳转
        if(!isNavigationFailure(err,NavigationFailureType.redirected) || prev!==START){
          this.ready = true
          this.readyErrorCbs.forEach(cb => cb(err))
        }
      }
    })
  }
  confirmTransition(route,onComplete,onAbort){
    const current = this.current
    this.pending = route
    const abort = err => {
      if(!isNavigationFailure(err) && isError(err)){
        if(this.errorCbs.length){
          this.errorCbs.forEach(cb => cb(err))
        }else{
          ...// navigationError
        }
      }
      onAbort && onAbort(err)
    }
    const lastRouteIndex = route.matched.length - 1
    const lastCurrentIndex = current.matched.length - 1
    // 跳转前后路由一致，abort重复跳转错误
    if(isSameRoute(route,current) && lastRouteIndex === lastCurrentIndex && route.matched[lastRouteIndex] === current.matched[lastCurrentIndex]){
      this.ensureURL()
      return abort(createNavigationDuplicatedError(current,route))
    }
    // 当前路由和跳转路由的统一的部分/当前路由不同/跳转路由的不同
    const { updated, deactivated, activated } = resolveQueue(this.current.matched,route.matched)
    // 获取beforeRouterLeave路由守卫/beforeRouteUpdate守卫/解析组件/获取路由跳转的守卫执行方法
    const queue = [].concat(extractLeaveGuards(deactivated),this.router.beforeHooks,extractUpdatedHooks(updated),activated.map(m => m.beforeEnter),resolveAsyncComponents(activated))
    const iterator = (hook,next) => {
      // 跳转失败
      if(this.route !== pending){
        return abort(createNavigationCancelledError(current,route))
      }
      try{
        // 执行路由守卫/路由组件解析等
        hook(route,current,(to) => {
          // next传入false/或者Error不执行跳转
          if(to === false){
            this.ensureURL(true)
            abort(createNavigationAbortedError(current,to))
          }else if(isError(to)){
            this.ensureURL(true)
            abort(to)
          }else if(typeof to === 'string' || (typeof to === 'object' && typeof to.path === 'string' || typeof to.name === 'string')){
            // next传入路由，爆出重定向错误，但是执行跳转
            abort(createNavigationRedirectedError(current,route))
            // 执行跳转
            if(typeof to === 'object' && to.replace){
              this.replace(to)
            }else{
              this.push(to)
            }
          }else{
            // 否则执行下一个路由守卫或者解析
            next(to)
          }
        })
      }catch(e){
        abort(e)
      }
    }
    // 依此执行路由变换，路由解析完成执行回调跳转，进入路由进入解析
    runQueue(queue,iterator,() => {
      // 获取beforeRouteEnter路由守卫
      const enterGuards = extractEnterGuards(activated)
      // 获取路由组件解析钩子函数
      const queue = enterGuards.concat(this.router.resolveHooks)
      // 依此执行路由变换，路由解析完成执行回调跳转，进入路由跳转完成
      runQueue(queue,iterator,() => {
        if(this.pending !== route){
          return abort(createNavigationCancelError(current,route))
        }
        this.pending = null
        onComplete(route)
        if(this.router.app){
          this.router.app.$nextTick(() => {
            handleRouterEntered(route)
          })
        }
      })
    })
  }
  updateRoute(route){
    // 路由更新
    this.current = route
    this.cb && this.cb(route)
  }
  setupListeners(){}
  teardown(){
    // 清除路由监听/路由状态回归初始化
    this.listeners.forEach(cleanuplistener => {
      cleanuplistener()
    })
    this.listeners = []
    this.current = START
    this.pending = null
  }
}
```
#### normalizeBase

```js
function normalizeBase(base){
  if(!base){
    // 初始化base路径
    if(inBrowser){
      const baseEl = document.querySelector('.base')
       base = ( baseEl && baseEl.getAttribute('href')) || '/'
       base = base.replace(/^https?:\/\/[^\/]+/,'')
    }else{
      base = '/'
    }
  }
  // 判断路径初始是否以/开头
  if(base.charAt(0) !== '/'){
    base = '/' + base
  }
  return base.replace(/\/$/,'')
}
```

#### resolveQueue

```js
// 比对跳转路由的不同
function resolveQueue(current,next){
  let i
  const max = Math.max(current.length,next.length)
  // 当路由路径出现不同
  for(i = 0; i < max; i ++){
    if(current[i] !== next[i]){
      break
    }
  }
  return {
    updated:next.slice(0,i), // 原路径和跳转路径的相同
    activated:next.slice(i), // 跳转路径的不同
    deactivated:current.slice(i) // 原路径不同
  }
}
```

#### extractLeaveGuards

```js
function extractLeaveGuards(deactivated){
  // 获取离开当前路径的路由守卫
  return extractGuards(deactivated,'beforeRouteLeave',bindGuard,true)
}
```

##### extractGuards

```js
function extractGuards(records,name,bind,reverse){
  // 展开路由的组件
  const guards = flatMapComponents(record,(def,instance,match,key) => {
    // 获取组件中的路由守卫
    const guard = extractGuard(def,name)
    if(guard){
      return Array.isArray(guard) ? guard.map(guard => bind(guard,instance,match,key)) : bind(guard,instance,match,key)
    }
    // 展开路由守卫
    return flatten(reverse ? guards.reverse() : guards)
  })
}
```

##### extractGuard

```js
// 获取组件中的路由守卫方法
function extractGuard(def,key){
  if(typeof def !== 'function'){
    def = _Vue.extend(def)
  }
  return def.options[key]
}
```

##### bindGuard

```js
function bindGuard(guard,instance){
  // 闭包返回组件的路由守卫执行
  if(instance){
    return function boundRouteGurad(){
      return guard.apply(instance,arguments)
    }
  }
}
```

##### extractEnterGuards

```js
function extractEnterGuards(activated){
  // 获取beforeRouteEnter路由守卫
  return extractGuards(activated,'beforeRouteEnter',(guard,_,match,key) => bindEnterGuard(guard,match,key))
}
```

##### bindEnterGuard

```js
function bindEnterGuard(guard,match,key){
  return function routeEnterGuard(to,from,next){
    // 获取enterCbs的回调
    return guard(to,from,cb => {
      if(typeof cb === 'function'){
        if(!match.enterCbs[key]){
          match.enterCbs[key] = []
        }
        match.enterCbs[key].push(cb)
      }
      next(cb)
    })
  }
}
```

#### runQueue

```js
function runQueue(queue,fn,cb){
  const step = index => {
    // 序列方法执行完成，即执行回调
    if(index >= queue.length){
      cb()
    }else{
      // 判断路由守卫是否存在
      if(queue[index]){
      // 执行路由守卫
        fn(queue[index],() =>{
          step(index + 1)
        })
      }else{
        step(index + 1)
      }
    }
  }
  step(0)
}
```
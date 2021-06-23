---
title: VueRouter源码解析五
date: 2021-06-22 23:04:00
tags:
---

### HTML5History

```js
class HTML5History extends History{
  _startLocation;
  constructor(router,base){
    super(router,base)
    // 获取html5路由初始
    this._startLocation = getLocation(this.base)
  }
  setupListeners(){
    if(this.listeners.lenth > 0){
      return
    }
    // 当前路由实例
    const router = this.router
    // 路由切换的滚动行为
    const expectScroll = router.options.scrollBehavior
    //  判断是否支持historyPushState 和 滚动行为是否存在
    const supportsScroll = supportsPushState && expectScroll 
    if(supportScroll){
      // 记录当前滚动位置
      this.listeners.push(setupScroll())
    }
    const handleRoutingEvent = () => {
      const current = this.current
      const location = getLocation(this.base)
      // 当前处于路由初始状态
      if(this.current === START && location === this._startLocation){
        return
      }
      // 跳转路由
      this.transitionTo(location,route => {
        // 跳转后滚动
        if(supportScroll){
          handleScroll(router,route,current,true)
        }
      })
    }
    window.addEventListener('popstate',handleRoutingEVent)
    this.listeners.push(() => {
      window.removeEventListener('popstate',handleRoutingEvent)
    })
  }
  // 执行跳转
  go(n){
    window.history.go(n)
  }
  push(location,onComplete,onAbort){
    const { current: fromRoute } = this
    // 执行跳转
    this.transitionTo(location,route => {
      // 更新浏览记录
      pushState(cleanPath(this.base+ route.fullPath))
      // 执行滚动
      handleScroll(this.router,route,fromRoute,false)
      onComplete && onComplete(route)
    },onAbort)
  }
  replace(location,onComplete,onAbort){
    const { current: fromRoute } = this
    // 执行跳转
    this.transitionTo(location,route => {
      // 替换浏览记录
      replaceState(cleanPath(this.base + route.fullPath))
      // 执行滚动
      handleScroll(this.router,route,fromRoute,false)
      onComplete && onComplete(route)
    },onAbort)
  }
  ensureURL(push){
    // 判断当前路径是否没有跳转进行跳转
    if(getLocation(this.base) !== this.current.fullPath){
      const current = cleanPath(this.base+ this.current.fullPath)
      push ? pushState(current) : replaceState(current)
    }
  }
  // 获取当前路径
  getCurrentLocation(){
    return getLocation(this.base)
  }
}
```

#### setupScroll

```js
function setupScroll(){
  // 利用浏览器特性回到上一个页面的滚动位置，auto滚动 manual不滚动
  if('scrollRestoration' in window.history){
    window.history.scrollRestoration = 'manual'
  }
  // 获取域名
  const protocolAndPath = window.location.protocol + '//' + window.location.host
  // 获取绝对路径
  const absolutePath = window.location.href.replace(protocolAndPath,'')
  // 复制浏览器记录
  const stateCopy = extend({},window.history.state)
  stateCopy.key = getStateKey()
  // 以当前链接替换浏览器记录
  window.history.replaceState(stateCopy,'',absolutePath)
  // 监听页面改变触发记录滚动位置
  window.addEvenetListener('popstate',handlePopState)
  return () => {
    window.removeEventListener('popstate',handlePopState)
  }
}
```

##### getStateKey

```js
const Time = inBrowser && window.performance && window.performance.now ? window.performance : Date
const  genStateKey = () => Time.now().toFixed(3) // 获取当前时间
let _key = genStateKey()
const getStateKey = () => _key // 获取当前时间作为key
const setStateKey = (key) => _key = key
```

##### handlePopState

```js
function handlePopState(e){
  // 记录位置
  saveScrollPosition()
  // 记录页面时间
  if(e.state && e.state.key){
    setStateKey(e.state.key)
  }
}
const positionStore = Object.create(null)
function saveScrollPosition(){
  // 记录当前时间点的滚动位置
  const key = getStateKey()
  if(key){
    positionStore[key] = {x:window.pageXOffset,y:window.pageYOffset}
  }
}
```

##### handleScroll

```js
function handleScroll(router,to,from,isProp){
  // 没有挂载vue实例
  if(!router.app){
    return
  }
  // 判断没有滚动行为
  const behavior = router.options.scrollBehavior
  if(!behavior){
    return
  }
  ...// 滚动行为必须是个函数
  router.app.$nextTick(() => {
    // 获取当前的滚动位置
    const position = getScrollPosition()
    // 返回应该滚动到的位置
    const shouldScroll = behavior.call(router,to,from,isProp ? position : null)
    if(!shouldScroll) return
    // 执行滚动
    if(typeof shouldScroll.then === 'function'){
      shouldScroll.then(shouldScroll => {
        scrollPosition(shouldScroll,position)
      }).catch(err => {
        if(process.env.NODE_ENV !== 'production'){
          assert(false,err.toString)
        }
      })
    }else{
      scrollPosition(shouldScroll,position)
    }
  })

}
```

##### scrollPosition

```js
function scrollPosition(shouldScroll,position){
  const isObject = typeof shouldScroll === 'object'
  // 获取位置
  if(isObject && typeof shouldScroll.selector === 'string'){
    // 获取dom
    const el = hashStartsWithNumberRE.test(shouldScroll.selector) ? document.getElementById(shouldScroll.selector.slice(1)) : document.querySelector(shouldScroll.selector)
    if(el){
      let offset = shouldScroll.offset && typeof shouldScroll.offset === 'object' ? shouldScroll.offset : {}
      offset = normalizeOffset(offset)
      position = getElementPosition(el,offset)
    }else if(isValidPosition(shouldScroll)){
      position = normalizePosition(shouldScroll)
    }
  }else if(isObject && isValidPosition(shouldScroll)){
    position = normalizePosition(shouldScroll)
  }
  // 执行滚动行为
  if(position){
    if('scrollBehavior' in document.documentElement.style){
      window.scrollTo({
        left: position.x,
        top: position.y,
        behavior: shouldScroll.behavior
      })
    }else{
      window.scrollTo(position.x,position.y)
    }
  }
}
```

#### getLocation

```js
function getLocation(base){
  let path = window.location.pathname
  // 获取base之外的路径
  if(base && path.toLowerCase().indexOf(base.toLowerCase()) === 0){
    path = path.slice(base.length)
  }
  // 返回当前location
  return (path || '/') + window.location.search + window.location.hash
}
```

#### supportPushState

```js
// 判断系统版本是否支持pushState
const supportPushStat = inBrowser && (function(){
  const ua = window.navigator.userAgent
  if( (ua.indexOf('Android 2.')!== -1 || ua.indexOf('Android 4.0') !== -1)&& ua.indexOf('Mobile Safari') !== -1 && ua.indexOf('chrome') === -1 && ua.indexOf('WindowsPhone') ===) return false
  return window.history && typeof window.history.pushState === 'function'
})()
```

##### pushState

```js
function pushState(url,replace){
  // 存储当前滚动位置
  saveScrollPosition()
  const history = window.history
  // 执行跳转
  try{
    if(replace){
      const stateCopy = extend({},history.state)
      stateCopy.key = getStateKey()
      // 替换浏览记录同时更改记录的时间
      histroy.replaceState(stateCopy,'',url)
    }else{
      // 更新浏览记录
      history.pushState({key:setStateKey(getStateKey())},'',url)
    }
  }catch(e){
    window.location[replace ? 'replace' : 'assign'](url)
  }
}
```

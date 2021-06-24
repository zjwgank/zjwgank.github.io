---
title: VueRouter源码解析六
date: 2021-06-24 21:50:06
tags:
---

### HashHistory

```js
class HashHistory extends History{
  constructor(router,base,fallback){
    super(router,base)
    if(fallback && checkFallBack(this.base)){
      return
    }
    // 初始化切换路由
    ensureSlash()
  }
  setupListeners(){
    if(this.listeners.length > 0){
      return
    }
    const router = this.router
    const expectScroll = router.options.scrollBehavior
    const supportsScroll = supportsPushState && expectScroll
    // 判断添加了滚动设置和支持滚动行为，添加页面切换记录滚动位置
    if(supportsScroll){
      this.listeners.push(setupScroll())
    }
    const handleRoutingEvent = () => {
      const current = this.current
      // 判断当前为hash地址切换路径后直接返回
      if(!ensureSlash()){
        return
      }
      // 否则切换路由
      this.transitionTo(getHash(),route => {
        // 切换后页面滚动
        if(supportsScroll){
          handleScroll(this.router,route,current,true)
        }
        // 如果不支持pushState/ hash切换
        if(!supportsPushState){
          replaceHash(route.fullPath)
        }
      })
    }
    // 判断是否支持pushState，路径变化监听popstate时间还是hash事件
      const eventType = supportsPushState ? 'popstate' : 'hashchange'
      // 路径变化，handleRoutingEvent
      window.addEventListener(eventType,handleRoutingEvent)
      this.listeners.push(() => {
        window.removeEventListener(eventType,handleRoutingEvent)
      })
  }
  push(location,onComplete,onAbort){
    const { current: fromRoute} = this
    // 切换路由/路由守卫事件/页面滚动
    this.transitionTo(location,route => {
      pushHash(route.fullPath)
      handleScroll(this.router,route,fromRoute,false)
      onComplete && onComplete(route)
    },onAbort)
  }
  replace(location,onComplete,onAbort){
    const { current: fromRoute } = this
    //执行路由守卫/切换路由/页面滚动
    this.transitionTo(location,route => {
      replaceHash(route.fullPath)
      handleScroll(this.router,route,fromRoute,false)
      onComplete && onComplete(route)
    },onAbort)
  }
  go(n){
    window.history.go(n)
  }
  ensureURL(push){
    // 判断跳转后的路径和路由对象是否一致，不一致重新跳转
    const current = this.current.fullPath
    if(getHash() !== current)
    push ? pushHahs(current) : replaceHash(current)
  }
  getCurrentLocation(){
    // 获取当前路径
    return getHash()
  }
}
```

#### checkFallBack

```js
function checkFallBack(base){
  // 获取基于base的路径
  const location = getLocation(base)
  // 判断路径不以/#开头
  if(!/^\/#/.test(location)){
    // 替换当前location为/base/#/路径
    window.location.replace(cleanPath(base + '/#' + location))
    return true
  }
}
```

#### ensureSlash

```js
function ensureSlash(){
  // 获取当前路径的hash值
  const path = getHash()
  if(path.charAt(0) === '/'){
    return true
  }
  replaceHash('/'+hash)
  return false
}
```

##### getHash

```js
function getHash(){
  // 获取hash
  let href = window.location.href
  const index = href.indexOf('#')
  if(index < 0) return ''
  href = href.slice(index + 1)
  return href
}
```

##### replaceHash

```js
function replaceHash(path){
  // 路由切换
  if(supportsPushState){
    replaceState(path)
  }else{
    window.location.replace(getUrl(path))
  }
}
```

##### getUrl

```js
function getUrl(path){
  // 生成当前路径下的hash地址
  const href = window.location.href
  const i = href.indexOf('#')
  const base = i >= 0 ? href.slice(0,i) : href
  return `${base}#${path}`
}
```

#### pushHash

```js
function pushHash(path){
  // 切换路径
  if(supportsPushState){
    pushState(getUrl(path))
  }else{
    window.location.hash = hash
  }
}
```
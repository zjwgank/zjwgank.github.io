---
title: VueRouter源码解析七
date: 2021-06-24 22:48:12
tags:
---

### AbstractHistory

```js
class AbstractHistroy extends Histroy{
  index;
  stack;
  constructor(router,base){
    super(router,base)
    // 存储路由
    this.stack = []
    this.index = -1
  }
  push(location,onComplete,onAbort){
    // 执行路由守卫事件/组件渲染等
    this.transitionTo(location,route => {
      // 更新stack路由栈
      this.stack = this.stack.slice(0,this.index + 1).concat(route)
      this.index ++
      onComplete && onComplete(route)
    },onAbort)
  }
  replace(loaction,onComplete,onAbort){
    // 执行路由守卫事件/组件渲染等
    this.transitionTo(location,route => {
      // 替换当前路由
      this.stack = this.stack.slice(0,this.index).concat(route)
      onComplete && onComplete(route)
    },onAbort)
  }
  go(n){
    const targetIndex = this.index + n
    // 路由栈为空/跳转的路由不存在
    if(targetIndex < 0 || targetIndex >= this.stack.lenth){
      return 
    }
    // 获取要跳转的路由
    const route = this.stack[targetIndex]
    // 执行路由守卫/组件解析
    this.confirmTransitionTo(route,() => {
      const prev = this.current
      this.index = targetIndex
      // 更新路由/更新index/执行路由后置守卫
      this.updateRoute(route)
      this.router.afterHooks.forEach(hook => {
        hook && hook(route,prev)
      })
    }, err => {
      // 判断路由未切换
      if(isNavigationFailure(err,NavigationFailureType.duplicated)){
        this.index = targetIndex
      }
    })
  }
  // 获取当前路径
  getCurrentLocation(){
    const current = this.stack[this.stack.length - 1]
    return current ? current.fullPath: '/'
  }
  ensureURL(){}
}
```
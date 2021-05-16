---
title: Vue-nextTick
date: 2021-05-16 23:31:38
tags:
---

### nextTick

```js
// 标记是否启用微任务
export let isUsingMicroTask = false
// 存储nextTick的方法
const callback = []
let pending = false
// 执行nextTick的方法,同时清空nextTick的任务队列
function flushCallbacks(){
  pending = false
  const copies = callback.slice(0)
  callback.length = 0
  for(let i = 0;i < copies.length; i ++){
    copies[i]()
  }
}

let timeFunc

if(typeof Promise !== 'undefined' && isNative(Promise)){
  // 判断window环境/Node环境,Promise是否存在
  const p = Promise.resolve()
  // 依据Promise执行微任务调用nextTick
  timeFunc = () => {
    p.then(flushCallbacks)
    if(isIOS) setTimeout(noop)
  }
  // 标记启用微任务
  isUseingMicroTask = true
}else if(!isIE && typeof MutationObserver !== 'undefined' && (isNative(MutationObserver) || MutationObserver.toString() === '[object MutationObserverConstructor]')){
  // 标记当前非IE环境,MutationObserver是否存在
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  // 监控textNode,来调用nextTick
  observer.observe(textNode,{
    characterData: true
  })
  // textNode变化
  timeFunc = () => {
    counter = (counter + 1)%2
    textNode.data = String(counter)
  }
  // 标记启用微任务
  isUseringMicroTask = true
}else if(typeof setImmediate !== 'undefined' && isNative(setImmediate)){
  // 判断当前Node环境,setImmediate存在
  // 宏任务调用nextTick
  timeFunc = () => {
    setImmediate(flushCallbacks)
  }
}else {
  // 宏任务调用nextTick
  timeFunc = () => {
    setTimeout(flushCallbacks,0)
  }
}
// nextTick运行
export function nextTick(cb,ctx){
  let _resolve
  // 向nextTick的任务队列中存储任务
  callbacks.push(() => {
    if(cb){
      try{
        cb.call(ctx)
      }catch(e){
        handleError(e,ctx,'nextTick')
      }
    }else if(_resolve){
      _resolve(ctx)
    }
  })
  // 判断当前是否在执行nextTick,否则执行
  if(!pending){
    pending = true
    timeFunc()
  }
  // 判断当前不存在任务,返回调用对象
  if(!cb && typeof Promise !== 'undefined'){
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```
---
title: Vue源码解析八
date: 2021-05-07 22:22:01
tags:
---
```js
// Vue构造函数
function Vue(options){
  if(process.env.NODE_ENV !== 'production' && !(this instanceof Vue)){
    warn('Vue is a constructor ans shold be called with the `new` keyword')
  }
  this._init(options)
}
// 初始化设置
initMixin(Vue)
// 设置属性获取和方法
stateMixin(Vue)
// 事件方法设置
eventsMixin(Vue)
// 设置生命周期方法
lifecycleMixin(Vue)
//  $nextTick/render
renderMixin(Vue)

export default Vue
```

#### initMixin

```js
let uid = 0
export function initMixin(Vue){
  Vue.prototype._init = function(options){
    const vm = this
    vm._uid = uid ++
    let startTag,endTag
    // dev环境开启性能监控,记录初始化阶段
    if(process.env.NODE_ENV !== 'production' && config.performance && mark){
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }
    vm._isVue = true
    if(options && options._isComponent){
      // 将options的配置和Vue原型配置合并(作为组件)
      initInternalComponent(vm,options)
    }else{
      // 解析Vue原型链上的配置,将传入的options合并至Vue原型上的配置
      vm.$options = mergeOptions(resolveConstructorOptions(vm.constructor),options || {},vm)
    }
    // 判断环境为vm._renderProxy设置vm对象或proxy
    if(process.env.NODE_ENV !== 'production'){
      initProxy(vm)
    }else{
      vm._renderProxy = vm
    }
    vm._self = vm
    // 初始化vm的生命周期状态和观察器/ref/子元素
    initLifyCycle(vm)
    // 初始化事件
    initEvents(vm)
    // 初始化renderVnode,重定义attrs和listener的读写
    initRender(vm)
    callHook(vm,'beforeCreate')
    // 重定义inject的读取
    initInjections(vm)
    // 初始化props/methods/data/watch/computed/
    initState(vm)
    // 初始化provide
    initProvide(vm)
    callHook(vm,'created')

    if(process.env.NODE_ENV !== 'production' && config.performance && mark){
      vm._name = formatComponentName(vm,false)
      mark(endTag)
      // dev环境,开启性能监控,记录组件的渲染
      measure(`vue ${vm._name} init`,startTag,endTag)
    }
    // 生成dom挂载到界面
    if(vm.$options.el){
      vm.$mount(vm.$options.el)
    }
  }
}
```

#### stateMixin

```js
export function stateMixin(Vue){
  // 设置$data/$props
  const dataDef = {}
  dataDef.get = function(){ return this._data }
  const propsDef = {}
  propsDef.get = function(){ return this._props }
  ...
  Object.defineProperty(Vue.property,'$data',dataDef)
  Object.defindeProperty(Vue.property,'$props',propsDef)
  // 设置$set/$del
  Vue.prototype.$set = set
  Vue.prototype.$del = del
  // 设置$watch
  Vue.prototype.$watch = function(expOrFn,cb,options){
    const vm = this
    // 判断回调函数如果是对象,通过createWatcher重新$watch
    if(isPlainObject(cb)){
      return createWatcher(vm,expOrFn,cb,options)
    }
    options = options || {}
    options.user = true
    // 创建watcher对象
    const watcher = new Watcher(vm,expOrFn,cb,options)
    // immediate立即执行回调
    if(options.immediate){
      pushTarget()
      try{
        cb.call(vm,watcher.value)
      }catch(error){
        handleError(error,vm,`callback for immediate watcher "${watcher.expression}"`)
      }
      popTarget()
    }
    // 返回函数解除监听器操作
    return function unwatchFn(){
      watcher.teardown()
    }
  }
}
```

#### eventsMixin

```js
export function eventsMixin(Vue){
  const hookRE = /^hook:/
  // 设置$on属性操作,为vue的事件绑定回调方法
  Vue.prototype.$on = function(event,fn){...}
  // 设置$once,挂载on回调,执行时注销事件,执行方法
  Vue.prototype.$once = function(event,fn){
    const vm = this
    function on(){
      vm.$off(event,on)
      fn.apply(vm,argument)
    }
    on.fn = fn
    vm.$on(event,on)
    return vm
  }
  // $off,注销event事件的方法,如果不传入参数,则清空所有事件
  Vue.prototype.$off = function(event,fn){...}
  // $emit,触发event对应的回调,如果回调事件出错则catch返回
  Vue.prototype.$emit = function(event){
    const vm = this
    ...
    let cbs = vm._events[event]
    if(cbs){
      cbs = cbs.length > 1 ? toArray(cbs) : cbs
      const args = toArray(arguments,1)
      const info = `event handler for ${event}`
      for(let i = 0,l = cbs.length; i < l; i ++){
        invokeWithErrorHandling(cbs[i],vm,args,vm,info)
      }
    }
    return vm
  }
}
```

#### lifecycleMixin

```js
export function lifecycleMixin(Vue){
  Vue.prototype._update = function(vnode,hydrating){
    const vm = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const restoreActiveInstance = setActiveInstance(vm)
    vm._vnode = vnode
    // 比对node,更新dom
    if(!prevVnode){
      vm.$el = vm.__patch__(vm.$el,vnode,hydrating,false)
    }else{
      vm.$el = vm.__patch__(prevVnode,vnode)
    }
    restoreActiveInstance()
    // 更新__vue__
    if(prevEl){
      prevEl.__vue__ = null
    }
    if(vm.$el){
      vm.$el.__vue__ = vm
    }
    // 高阶组件判断
    if(vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode){
      vm.$parent.$el = vm.$el
    }
  }
  // 根据vue的监听器,判断是否要进行事件/computed/watch等更新
  Vue.prototype.$forceUpdate = function(){
    const vm = this
    if(vm._watcher){
      vm._watcher.update()
    }
  }
  Vue.prototype.$destroy = function(){
    const vm = this
    if(vm._isBeingDestroyed){
      return
    }
    callHook(vm,'beforeDestroy')
    vm._isBeingDestroyed = true
    const parent = vm.$parent
    // 移除父组件下的当前元素
    if(parent && !parent._isBeingDestroyed && !vm.$options.abstract){
      remove(parent.$children,vm)
    }
    // 移除监听器
    if(vm._watcher){
      vm._watcher.teardown()
    }
    let i = vm._watchers.length
    while(i --){
      vm._watchers[i].teardown()
    }
    // 移除data监听
    if(vm._data.__ob__){
      vm._data.__ob__.vmCount --
    }
    vm._isDestroyed = true
    // 重新比对,dom渲染
    vm.__patch__(vm._vnode,null)
    callHook(vm,'destroyed')
    // 移除事件
    vm.$off()
    if(vm.$el){
      vm.$el.__vue__ = null
    }
    if(vm.$vnode){
      vm.$vnode.parent = null
    }
  }
}
```

#### renderMixin

```js
export function renderMixin(Vue){
  // 在Vue属性上挂载各种方法
  installRenderHelpers(Vue.prototype)
  // 判断环境采用宏任务实现微任务
  Vue.prototype.$nextTick = function(fn){
    return nextTick(fn,this)
  }
  // 生成vnode
  Vue.prototype._render = function(){
    cosnt vm = this
    const { render, _parentVnode } = vm.$options
    // 获取所有的插槽
    if(_parentVnode){
      vm.$scopedSlots = normalizeScopedSlots(_parentVnode.data.scopedSlots,vm.$slots,vm.$scopedSlots)
    }
    vm.$vnode = _parentVnode
    let vnode
    try{
      currentRenderingInstance = vm
      // 根据render生成vnode
      vnode = render.call(vm._renderProxy,vm.$createElement)
    }catch(e){
      hanleError(e,vm,'render')
      if(process.env.NODE_ENV !== 'production' && vm.$options.renderError){
        try{
          // 生成报错vnode
          vnode = vm.$options.renderError.call(vm._renderProxy,vm.$createElement,e)
        }catch(e){
          handleError(e,vm,'renderError')
          vnode = vm._vnode
        }
      }else{
        vnode = vm._vnode
      }
    }finally{
      currentRenderingInstance = null
    }
    // 返回vnode
    if(Array.isArray(vnode) && vnode.length === 1){
      vnode = vnode[0]
    }
    if(!vnode instanceof VNode){
      if(process.env.NODE_ENV !== 'production' && Array.isArray(vnode)){
        warn(
          'Multiple root nodes returned from render function. Render function should return a single root node',vm
        )
      }
      vnode = creatEmptyVNode()
    }
    vnode.parent = _parentVnode
    return vnode
  }
}
```
---
title: Vue源码解析二
date: 2021-04-21 22:29:50
tags:
---
```js
Vue.prototype.$mount = function(el,hydrating){
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this,el,hydrating)
}
```
Vue实例上绑定的`$mount`方法

1. 首先通过`el`和`const inBrowser = typeof window !== 'undefined'`,判断当前是否在浏览器环境进行获取`query(el) => el ? docuemnt.querySelector(el) : document.createElement('div')`
2. 执行`mountComponent(this,el,hydrating)`,`this`指向方法的调用者,`el`指向挂载的元素上,`hydrating`关联服务端渲染

```js
export function mountComponent(vm,el,hydrating){
  vm.$el = el; // 将元素挂载到vue实例上
  if(!vm.options.render){
    vm.options.render = createEmptyVNode // 当没有render函数时,将创建VNode赋值给render
    if(process.env.NODE_ENV !== 'production'){
      ...// 针对非production环境输出警告
    }
  }
  callHook(vm,'beforeMount') // 调用beforeMount生命周期钩子
  let updateComponent // 核心方法
  if(process.env.NODE_ENV !== 'production' && config.performance && mark){
      // 开启性能监控
    updateComponent = () => {
      const name = vm._name
      const id = vm._uid
      const startTag = `vue-perf-start:${id}`
      const endTag = `vue-perf-end:${id}`
      mark(startTag)
      const vnode = vm._render() // 生成虚拟节点VNODE
      mark(endTag)
      measure(`vue ${name} render`,startTag,endTag) // 记录render时间
      mark(startTag)
      vm._update(vnode,hydrating) // 虚拟节点转化为Dom
      mark(endTag)
      measure(`vue ${name} patch`,startTag, endTag) // 记录更新时间
    }
  }else{
    updateComponent = () => {
      vm._update(vm._render(),hydrating)
    }
  }
  new Watcher(vm,updateComponent,noop,{ // 执行vm._render和vm._update
    before(){
      if(vm._isMounted && !vm._isDestroyed){
        callHook(vm,'beforeUpdate')
      }
    }
  },true)
  hydrating = false
  if(vm.$vnode == null){
    // 当虚拟节点都转换为DOM时,表示dom挂载结束,调用mounted生命周期钩子
    vm._isMounted = true
    callHook(vm,'mounted')
  }
  return vm
}
```

#### vm._render

```js
Vue.prototype._render = function(){
  const vm = this
  const { render, _parentVnode } = vm.$options
  if(_parentVnode){ // 添加虚拟父dom处理
    vm.$scopedSlots = normalizeScopedSlots(_parentVnode.data.scopedSlots,vm.$slots,vm.$scopedSlots)
  }
  vm.$vnode = _parentVnode
  let vnode
  try{
    currentRenderingInstance = vm
    // 传入vm.$createElement方法,生成vnode
    vnode = render.call(vm._renderProxy,vm.$createElement)
  }catch(e){
    ...// 生成错误信息展示
  }finally{
    currentRenderingInstance = null
  }
  if(Array.isArray(vnode) && vnode.length === 1){
    vnode = vnode[0]
  }
  if(!(vnode instanceof VNode)){
    ... // 添加vnode非虚拟dom处理
  }
  vnode.parent = _parentNode
  return vnode
}
```

#### vm._update

```js
Vue.prototype._update = function(vnode,hydrating){
  const vm = this
  const prevEl = vm.$el
  const prevVnode = vm._vnode // 之前渲染的虚拟dom
  const restoreActiveInstance = setActiveInstance(vm)
  vm._vnode = vnode // 当前渲染的虚拟dom
  if(!prevVnode){
    // 如果之前不存在虚拟dom直接渲染在$el
    vm.$el = vm.__patch__(vm.$el,vnode,hydrating,false)
  }else{
    // 和之前的dom做比对,生成dom
    vm.$el = vm.__patch__(prevVnode,vnode)
  }
  // 生成dom后,切换实例
  restoreActiveInsatnce()
  if(prevEl){
    prevEl.__vue__ = null
  }
  if(vm.$el){
    // 在渲染的dom上挂载当前vue对象
    vm.$el.__vue__ = vm
  }
  if(vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode){
    vm.$parent.$el= vm.$el
  }
}
```

#### createEmptyVNode

```js
class VNode{
  ...
}
const createEmptyVNode = (text)=> {
  const node = new VNode()
  node.text = text
  node.isComment = true
  return node
}
```

#### callHook

```js
function callHook(vm,hook){
  pushTarget()
  const handlers = vm.$options[hook] // 获取hook阶段函数
  const info = `${hook} hook`
  if(handlers){ // 执行hook阶段函数,添加异常捕捉
    for(let i = 0,j = handlers.length;i < j; i ++){
      invokeWithErrorHandling(handlers[i],vm,null,vm,info)
    }
  }
  if(vm._hasHookEvent){
    vm.$emit('hook:'+hook) // 执行hook阶段
  }
  popTarget()
}
function invokeWithErrorHandling(handler,context,args,vm,info){
  ... // 为handler函数执行添加异常捕捉
}
```

### Watcher

```js
class Watcher{
  ...
  constructor(vm,expOrFn,cb,options,isRenderWatcher){
    this.vm = vm
    if(e){
      vm._watcher = this
    }
    vm._watchers.push(this)
    if(options){
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = !!options.before
    }else{
      this.deep = this.user = this.lazy = this.sync = false
    }
    ...
    if(typeof expOrFn === 'function'){
      this.getter = expOrFn
    }else{
      ...
    }
    this.value = this.lazy ? undefined : this.get()
  }

  get(){
    pushTarget()
    let value 
    const vm = this.vm
    try{
      value = this.getter.call(vm,vm)
    }.catch(e){
      ...
    }
  }
}
```
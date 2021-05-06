---
title: Vue源码解析七
date: 2021-04-30 21:00:55
tags:
---
```js
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'
import { isServerRendering } from 'core/util/env'
import { FunctionalRenderContext } from 'core/vdom/create-functional-component'
// 为Vue挂载各种方法
initGlobalAPI(Vue)
// 重新定义$isServer,$sseContext,FunctionalRenderContext
Object.defineProperty(Vue.prototype,'$isServer',{
  get:isServerRendering // 通过Vue.prototype.$isServer判断当前是否是ssr渲染
})
Object.defineProperty(Vue.prototype,'$ssrContext',{
  get(){ // 通过Vue.prototype.$ssrContext 获取当前的上下文
    return this.$vnode && this.$vnode.ssrContext
  }
})
// Vue.FunctionalRenderContext = {value:FunctionalRenderContext}
Object.defineProperty(Vue,'FunctionalRenderContext',{
  value:FunctionalRenderContext
})
Vue.version = '__VERSION__'
export default Vue
```

#### isServerRendering

```js
let _isServer
export const isServerRendering = () => {
  if(_isServer === undefined){
    // 标记非微信环境,非浏览器环境,node环境下,根据Vue的环境变量判断是否是ssr
    if(!isBrowser && !inWeex && typeof global !== undefined){
      _isServer = global['process'] && global['process'].env.VUE_ENV === 'server'
    }else{
      _isServer = false
    }
  }
}
```

#### FunctionalRenderContext

```js
export function FunctionalRenderContext(data,props,children,parent,Ctor){
  const options = Ctor.options
  let contextVm
  if(hasOwn(parent,'_uid')){
    contextVm = Object.create(parent)
    contextVm._original = parent
  }else{
    contextVm = parent
    parent = parent._original
  }
  const isCompiled = isTrue(options._compiled)
  const needNormalization = !isCompiled
  // 
  this.data = data
  this.props = props
  this.children = children
  this.parent = parent
  this.listeners = data.on || emptyObject
  // 挂载当前的注入
  this.injections = resolveInject(options.inject,parent)
  // 输出子组件上的所有插槽,如果插槽不存在,赋值后将所有插槽序列化到scopeSlots
  this.slots = () => {
    if(!this.$slots){
      // 将所有插槽序列化后保存到data.scopeSlots
      normalizeScopeSlots(
        data.scopeSlots,
        this.$slots = resolveSlots(children,parent) // 获取子组件上的所有插槽
        )
    }
    return this.$slots
  }
  // this.scopedSlots = 序列化插槽
  Object.define(this,'scopedSlots',({
    enumerable: true,
    get(){
      return normalizeScopedSlots(data.scopeSlots,this.slots())
    }
  })
  if(isCompiled){
    this.$options = options
    this.$slots = this.slots()
    this.$scopedSlots = normalizeScopedSlots(data.scopedSlots,this.$slots)
  }
  if(options._scopeId){
    this._c = (a,b,c,d)=>{
      // 输出虚拟节点
      const vnode = createElement(contextVm,a,b,c,d,needNormalization)
      if( vnode && !Array.isArray(vnode)){
        vnode.fnScopeId = options._scopeId
        vnode.fnContext = parent
      }
      return vnode
    }
  }else{
    this._c = (a,b,c,d) => createElement(contextVm,a,b,c,d,needNormalization)
  }
}
```

#### createElement

```js
export function createElement(context,tag,data,children,normalizationType,alwaysNormalize){
  if(Array.isArray(data) || isPrimitive(data)){
    normalizationType = children
    children = data
    data = undefined
  }
  if(isTrue(alwaysNormalize)){
    normalizationType = ALWAYS_NORMALIZE
  }
  return _createElement(context,tag,data,children,normalizationType)
}
export function _createElement(context,tag,data,children,normalizationType){
  if(isDef(data) && isDef(data.__ob__)){
    ... // 当构建vnode的数据是被observed,返回空vnode
  }
  // 当采用组件,采用is作为标签
  if(isDef(data) && isDef(data.is)){
    tag = data.is
  }
  // 如果标签不存在,返回空vnode
  if(!tag){
    return createEmptyVNode()
  }
  ... // 针对非原始值的key的处理
  // 支持子组件为function的操作
  if(Array.isArray(children) && typeof children[0] === 'function'){
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  // 序列化子节点
  if(normalizeType === ALWAYS_NORMALIZE){
    children = normalizeChildren(children)
  }else if(normalizeType === SIMPLE_NORMALIZE){
    children = simpleNormalizeChildren(children)
  }
  let vnode,ns
  if(typeof tag === 'string'){
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNameSpace(tag)
    if(config.isReservedTag(tag)){
      ...// 添加native修饰符只能用在component标签警告
      vnode = new VNode(config.parsePlatformTagName(tag),data,children,null,null,context)
    }else if((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options,'components',tag))){
      // 在引入的组件中查找是否存在该标签
      // 根据组件配置创建vnode
      vnode = createComponent(Ctor,data,context,children,tag)
    }else{
      // 创建vnode
      vnode = new VNode(tag,data,children,undefined,undefined,context)
    }
  }else{
    vnode = createComponent(tag,data,context,children)
  }
  if(Array.isArray(vnode)){
    return vnode
  }else if(isDef(vnode)){
    // 为vnode挂载ns,注册data上的style和class
    if(isDef(ns)) applyNs(vnode,ns)
    if(isDef(data)) registerDeepBindings(data)
    return vnode
  }else{
    // vnode不存在返回空节点
    return createEmptyVNode()
  }
}
```

#### createComponent

```js
export function createComponent(Ctor,data,context,children,tag){
  // 如果标签为null或undefined直接返回
  if(isUndef(Ctor)){
    return
  }
  const baseCtor = Ctor.$options._base
  if(isObject(Ctor)){
    Ctor = baseCtor.extend(Ctor)
  }
  // 判断如果Ctor不是构造函数或者组件工厂报错
  if(typeof Ctor !== 'function'){
    ...
  }
  let asyncFactory
  if(isUndef(Ctor.cid)){
    asyncFactory = Ctor
    // 返回组件的构造函数
    Ctor = resolveAsyncComponent(asyncFactory,baseCtor)
    if(Ctor === undefined){
      // 返回空节点
      return createAsyncPlaceholder(asyncFactory,data,context,children,tag,)
    }
  }
  data = data || {}
  // 解析Ctor的配置
  resolveConstructorOptions(Ctor)
  // 解析v-model,事件/属性/回调
  if(isDef(data.model)){
    transformModel(Ctor.options,data)
  }
  // 为data挂载prop
  const propsData = extractPropsFormVNodeData(data,Ctor,tag)
  if(isTrue(Ctor.options.functional)){
    // 生成函数式组件
    return createFunctionalComponent(Ctor,propsData,data,context,children)
  }
  // 定义挂载的事件/原生事件/插槽
  const listeners = data.on
  data.on = data.nativeOn
  if(isTrue(Ctor.options.abstract)){
    const slot = data.slot
    data = {}
    if(slot){
      data.slot = slot
    }
  }
  // 挂载组件的hook函数
  installComponentHooks(data)
  // 生成vnode
  const name = Ctor.options.name || tag
  const vnode = new VNode(`vue-component-${Ctor.cid}${name ? `-${name}`:''}`,data,undefined,undefined,undefined,context,{Ctor,propsData,listeners,tag,children},asyncFactory)
  ...
  return vnode
}
```

#### resolveInject

```js
export function resolveInject(inject,vm){
  if(inject){
    const result = Object.create(null)
    const keys = hasSymbol ? Reflect.ownKeys(inject) : Object.keys(inject)
    // 针对注入的处理
    for(let i = 0; i < keys.length; i ++){
      const key = keys[i]
      if(key === '__ob__') continue
      // 获取注入的父节点来源
      const provideKey = inject[key].from 
      let source = vm
      // 从父节点上获取注入的方法或者属性
      while(source){
        if(source._provided && hasOwn(source._provided,providedKey)){
          result[key] = source._provided[providedKey]
          break
        }
        source = source.$parent
      }
      // 如果父节点不存在,则根据注入的属性获取
      if(!source){
        if('default' in inject[key]){
          const provideDefault = inject[key].default
          result[key] = typeof provideDefault === 'function' ? provideDefault.call(vm) : provideDefault
        }else if(process.env.NODE_ENV !== 'production'){
          warn(`Injection "${key}" not found`,vm)
        }
      }
    }
    return result
  }
}
```
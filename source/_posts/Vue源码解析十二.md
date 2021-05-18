---
title: Vue源码解析十二
date: 2021-05-17 21:51:15
tags:
---

### VNode

```js
class VNode{
  ...prop
  constructor(tag,data,children,text,elm,context,componentOptions,asyncFactory){
    // 为prop赋值
    ...
  }
  // 获取componentInstance
  get child(){
    return this.componentInstance
  }
}
// vnode为我们提供空节点/文本节点/节点克隆的操作
```

### patch

```js
const emptyNode = new VNode('',{},{})
const hooks = ['create','activate','update','remove','destroy']
function createPatchFunction(backend){
  let i,j;
  const cbs = {}
  // 获取modules--属性/指令等操作,nodeOps--节点操作
  const { modules, nodeOps } = backend
  // 在cbs中存入节点上的钩子函数
  for(i = 0; i < hooks.length; i ++){
    cbs[hooks[i]] = {}
    for(j = 0; j < modules.length; j ++){
      if(isDef(modules[j][hooks[i]])){
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }
  ...
  let hydrationBailed = false
  const isRenderedModule = makeMap('attrs,class,staticClass,staticStyle,key')
  return function patch(oldVnode,vnode,hydrating,removeOnly){
    // 原vnode存在,当前vnode不存在
    if(isUndef(vnode)){
      // 清除oldVnode和oldVnode上挂载的属性/指令,同时如果存在子节点同样清除
      if(isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return 
    }
    let isInitialPatch = false
    const insertedVnodeQueue = []
    if(isUndef(oldVnode)){
      // oldVnode不存在,根据vnode是否存在创建新节点
      isInitialPatch = true
      // 创建节点,挂载属性/样式等,调用钩子函数,插入
      createElm(vnode,insertedVnodeQueue)
    }else{
      const isRealElement = isDef(oldVnode.nodeType)
      //  对比节点的子节点,进行更新/添加/移除操作
      if(!isRealElement && sameVnode(oldVnode,vnode)){
        patchVnode(oldVnode,vnode,insertedVnodeQueue,null,null,removeOnly)
      }else{
        if(isRealElement){
          //... ssr渲染
        }
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)
        // 创建新节点
        createElm(vnode,insertedVnodeQueue,oldElm._leaveCb? null : parentElm,nodeOps.nextSibling(oldElm))
        if(isDef(vnode.parent)){
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          // 更新父节点
          while(ancestor){
            for(let i = 0; i < cbs.destroy.length; ++ i){
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if(patchable){
              for(let i = 0; i < cbs.create.length; ++ i){
                cbs.create[i](emptyNode,ancestor)
              }
              const insert = ancestor.data.hook.insert
              if(insert.merged){
                for(let i = 1; i < insert.fns.length; i ++){
                  insert.fns[i]()
                }
              }
            }else{
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }
        // 删除旧节点
        if(isDef(parentElm)){
          removeVnodes([oldVnode],0,0)
        }else if(isDef(oldVnode.tag)){
          invokeDestroyHook(oldVnode)
        }
      }
    }
    invokeInsertHook(vnode,insertedVnodeQueue,isInitialPatch)
    return vnode.elm
  }
}
```

### create-functional-component

```js
function createFunctionalComponent(Ctor,propsData,data,contextVm,children){
  const options = Ctor.options
  const props = {}
  const propOptions = options.props
  // 创建props
  if(isDef(propOptions)){
    for(const key in propOptions){
      props[key] = validateProp(key,propOptions,propsData || emptyObject)
    }
  }else{
    if(isDef(data.attrs)) mergeProps(props,data.attrs)
    if(isDef(data.props)) mergeProps(props,data.props)
  }
  // 创建一个函数式组件上下文
  const renderContext = new FunctionalRenderContext(data,props,children,contextVm,Ctor)
  const vnode = options.render.call(null,renderContext._c,renderContext)
  if(vnode instanceof VNode){
    // 返回一个克隆节点
    return cloneAndMarkFunctionalResult(vnode,data,renderContext.parent,options,renderContext)
  }else if(Array.isArray(vnode)){
    // 返回一个克隆节点数组
    const vnodes = normalizeChildren(vnode) || []
    const res = new Array(vnodes.length)
    for(let i = 0; i < vnodes.length; i ++){
      res[i] = cloneAndMarkFunctionalResult(vnodes[i],data,renderContext.parent,options,renderContext)
    }
    return res
  }
}
```

### createElement

```js
function _createElement(context,tag,data,children,normalizationType){
  if(isDef(data) && isDef(data.__ob__)){
    // 避免用被观察的数据作为vnode
    return createEmptyVnode()
  }
  // component v-bind:is
  if(isDef(data) && isDef(data.is)){
    tag = data.is
  }
  if(!tag){
    return createEmptyVnode()
  }
  if(process.env.NODE_ENV !== 'production' && isDef(data) && isDef(data.key) && !isPrimitive(data.key)){
    ... // key要求原始值
  }
  // 获取插槽
  if(Array.isArray(children) && typeof children[0] === 'function'){
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  if(normalizationType === ALWAYS_NORMALIZE){
    children = normalizeChildren(children)
  }else if(normalizationType === SIMPLE_NORMALIZE){
    children = simpleNormalizeChildren(children)
  }
  let vnode,ns;
  if(typeof tag === 'string'){
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if(config.isReservedTag(tag)){
      // 预留标签
      ... // native 修饰符
      // 创建Vnode
      vnode = new VNode(config.parsePlatformTagName(ns),data,children,undefined,undefined,context)
    }else if((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options,'components',tag))){
      // 创建vnode
      vnode = createComponent(Ctor,data,context,children,tag)
    }else{
      // 创建vnode
      vnode = new VNode(tag,data,children,undefined,undefined,context)
    }
  }else{
    vnode = createComponent(tag,data,context,children)
  }
  // 返回vnode
  if(Array.isArray(vnode)){
    return vnode
  }else if(isDef(vnode)){
    if(isDef(ns)) applyNs(vnode,ns)
    if(isDef(data)) registerDeepBindings(data)
    return vnode
  }else{
    return createEmptyVNode()
  }

}
```

### createComponent

```js
function createComponent(Ctor,data,context,children,tag){
  //  获取Vue实例
  const baseCtor = context.$options._base 
  // 当前Vm上的属性和方法
  if(isObject(Ctor)){
    Ctor = baseCtor.extend(Ctor)
  }
  if(typeOf Ctor !== 'function'){
    if(process.env.NODE_ENV !== 'production'){
      warn(`Invalid Component definition:${String(Ctor)}`,context)
    }
    return
  }
  // asyncComponent
  let asyncFactory
  if(isUndef(Ctor.cid)){
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory,baseCtor)
    if(Ctor === undefined){
      return createAsyncPlaceholder(asyncFactory,data,context,children,tag)
    }
  }
  data = data || {}
  // 解析vm实例上的配置
  resolveConstructorOptions(Ctor)
  // 转换v-model
  if(isDef(data.model)){
    transformModel(Ctor.options,data)
  }
  // 矫正prop
  const propsData = extractPropsFormVnodeData(data,Ctor,tag)
  // 创建函数式组件
  if(isTrue(Ctor.options.functional)){
    return createFunctionalComponent(Ctor,propsData,data,context,children)
  }
  const listeners = data.on
  data.on = data.nativeOn
  // 移除属性
  if(isTrue(Ctor.options.abstract)){
    const slot = data.slot
    data = {}
    if(slot){
      data.slot = slot
    }
  }
  // 钩子函数
  installComponentHooks(data)
  const name = Ctor.options.name || tag
  // 生成vnode
  const vnode = new VNode(`vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,data,undefined,undefined,undefined,context,{Ctor,propsData,listeners,tag,children,},asyncFactory)
  if(__WEEX__ && isRecycleableComponent(vnode)){
    return renderRecycleableComponentTemplate(vnode)
  }
  return vnode
}
```

### transformModel --- v-model

```js
function transformModel(options,data){
  // 获取v-model对应的属性和事件字段
  const prop = (options.model && options.model.prop) || 'value'
  const event = (options.model && options.model.event) || 'input'
  // 获取v-model的值
  (data.attrs || (data.attrs = {}))[prop] = data.model.value
  const on = data.on || data.on = {}
  // 获取v-model的触发事件和回调
  const existing = on[event]
  const callback = data.model.callback
  // 在on中添加v-model事件
  if(isDef(existing)){
    if(Array.isArray(existing) ? existing.indexOf(callback) === -1 : existing !== callback){
      on[event] = [callback].concat(existing)
    }
  }else{
    on[event] = callback
  }


}
```
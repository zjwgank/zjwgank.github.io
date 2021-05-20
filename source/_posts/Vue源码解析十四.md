---
title: Vue源码解析十四
date: 2021-05-19 22:05:32
tags:
---

### merge-hook

```js
export function mergeVNodeHook(def,hookKey,hook){
  //  获取钩子函数
  if(def instanceof VNode){
    def = def.data.hook || def.data.hook = {}
  }
  let invoker
  const oldHook = def[hookKey]
  // 执行钩子函数,移除wrappedHook
  function wrappedHook(){
    hook.apply(this,arguments)
    remove(invoker.fns,wrappedHook)
  }
  // vnode上新增钩子函数
  if(isUndef(oldHook)){
    // 执行现有钩子函数,并删除
    invoker = createFnInvoker([wrappedHook])
  }else{
    // 如果原钩子函数存在且merged,将当前钩子函数存入fns
    if(isDef(oldHook.fns) && isTrue(oldHook.merged)){
      invoker = oldHook
      invoker.fns.push(wrappedHook)
    }else{
      // 否则创建fns
      invoker = createFnInvoker([oldHook,warppedHook])
    }
  }

  invoker.merged = true
  def[hookKey] = invoker
}
```

### extractPropsFormVNodeData

```js
export function extractPropsFormVNodeData(data,Ctor,tag){
  // 获取vm的Props
  const propsOptions = Ctor.options.props
  if(isUndef(propsOptions)){
    return
  }
  const res = {}
  const { attrs, props } = data
  if(isDef(attrs) || isDef(props)){
    for(const key in propsOptions){
      // 驼峰转换
      const altKey = hyphenate(key)
      ...// prop注意事项
      // 为res添加props或者attrs的属性,同时删除attrs中的属性存在的key或者altKey
      checkProp(res,props,key,altKey,true) || checkProp(res,attrs,key,altKey,false)
    }
  }
  return res
}
```

### updateListeners

```js
export function updateListeners(on,oldOn,add,remove,createOnceHandler,vm){
  let name,def,cur,old,event
  for(name in on){
    def = cur = on[name]
    old = oldOn[name]
    // 获取当前事件的属性
    event = normalizeEvent(name)
    if(isUndef(cur)){
      // 当前事件不存在
      process.env.NODE_ENV !== 'production' && warn(`Invalid handler for event "${event.name}":got ` + String(cur))
    }else if(isUndef(old)){
      // 之前事件不存在
      if(isUndef(cur.fns)){
        cur = on[name] = createFnInvoker(cur,vm)
      }
      if(isTrue(event.once)){
        cur = on[name] = createOnceHandler(event.name,cur,event.capture)
      }
      // 根据event.name重新添加事件
      add(event.name,cur,event.capture,event.passive,event.params)
    }else if(cur !== old){
      old.fns = cur
      on[name] = old
    }
  }
  for(name in oldOn){
    if(isUndef(on[name])){
      // 新事件不存在,则移除事件
      event = normalizeEvent(name)
      remove(event.name,oldOn[name],event.capture)
    }
  }
}
```

### normalizeChildren

```js
// 根据类型生成vnodelist
export function normalizeChildren(children){
  return isPrimitive(children) ? [createTextVNode(children)] : Array.isArray(children) ? normalizeArrayChildren(children) : undefined
}
// 文本节点转换为一个,非文本节点正常存储
function normalizeArrayChildren(children,nestedIndex){
  const res = []
  let i,c,lastIndex,last
  for(i = 0; i < children.length;  i++){
    c = children[i]
    if(isUndef(c) || typeof c === 'boolean') continue
    lastIndex = res.length - 1
    last = res[lastIndex]
    if(Array.isArray(c)){
      // 序列化子节点存入到res
      if(c.length > 0){
        c = normalizeArrayChildren(c,`${nestedIndex || ''}_${i}`)
        // 判断c完全是textNode,同时上一个节点是textNode,转化为text存入res
        if(isTextNode(c[0]) && isTextNode(last)){
          res[lastIndex] = createTextVNode(last.text + c[0].text)
          c.shift()
        }
        res.push.apply(res,c)
      }
    }else if(isPrimitive(c)){
      // textNode存入res
      if(isTextNode(last)){
        res[lastIndex] = createTextVNode(last.text + c)
      }else if(c !== ''){
        res.push(createTextVNode(c))
      }
    }else{
      if(isTextNode(c) && isTextNode(last)){
        res[lastIndex] = createTextVNode(last.text + c.text)
      }else{
        if(isTrue(children._isVList) && isDef(c.tag) && isUndef(c.key) && isDef(nestedIndex)){
          c.key = `_vlist${nestedIndex}_${i}__`
        }
        res.push(c)
      }
    }
  }
  return res
}
```

### resolveAsyncComponent

```js
export function resolveAsyncComponent(factory,baseCtor){
  // factory报错
  if(isTrue(factory.error) && isDef(factory.errorComp)){
    return factory.errorComp
  }
  // 获取vm实例
  if(isDef(factory.resolved)){
    return factory.resolved
  }
  // 获取当前操作的vm实例
  const owner = currentRenderingInstance
  // 存入owners
  if(owner && isDef(factory.owners) && factory.owners.indefOf(owner) === -1){
    factory.owners.push(owner)
  }
  // loading状态
  if(isTrue(factory.loading) && isDef(factory.loadingComp)){
    return factory.loadingComp
  }
  if(owner && !isDef(factory.owners)){
    const owners = factory.owners = [owner]
    let sync = true
    let timerLoading = null
    let timerTimeout = null
    // vm挂载事件,销毁时移除owner
    owner.$on('hook:destroyed',() => remove(owners,owner))
    // 强制渲染vm
    const forceRender = (renderCompleted) => {
      for(let i = 0; i < owners.length; i ++){
        owners[i].$forceUpdate()
      }
      // 如果渲染完成,清空vm
      if(renderCompleted){
        owners.length = 0
        if(timerLoading !== null){
          clearTimeout(timerLoading)
          timerLoading = null
        }
        if(timerTimeout !== null){
          clearTimeout(timerTimeout)
          timerTimeout = null
        }
      }
    }
    const resolve = once((res) => {
      factory.resolved = ensureCtor(res,baseCtor)
      if(!sync){
        forceRender(true)
      }else{
        owners.length = 0
      }
    })
    const reject = once((reason) => {
      process.env.NODE_ENV !== 'production' && warn(`Failed to resolve async component: ${String(factory)}` + (reason ? `\nReason:${reason}` : ''))
      if(isDef(factory.errorComp)){
        factory.error = true
        forceRender(true)
      }
    })
    // 判断res是否异步
    const res = factory(resolve,reject)
    if(isObject(res)){
      // 如果是一个Promise
      if(isPromise(res)){
        if(isUndef(factory.resolved)){
          res.then(resolve,reject)
        }
      }else if(isPromise(res.component)){
        res.component.then(resolve,reject)
        if(isDef(factory.error)){
          factory.errorComp = ensureCtor(res.error,baseCtor)
        }
        // 解析异步是否执行完成
        if(isDef(res.loading)){
          factory.loadingComp = ensureCtor(res.loading,baseCtor)
          // 判断是否延时渲染
          if(res.delay === 0){
            factory.loading = true
          }else{
            timerLoading = setTimeout(() => {
              timerLoading = null
              // 延时执行后没有执行完成,判定仍处于loading状态,渲染dom
              if(isUndef(factory.resolved) && isUndef(factory.error)){
                factory.loading = true
                forceRender(false)
              }
            },res.delay || 200)
          }
        }
        // 判断解析超时,reject
        if(isDef(res.timeout)){
          timerTimeout = setTimeout(() => {
            timerTimeout = null
            if(isUndef(factory.resolved)){
              reject(process.env.NODE_ENV !== 'production' ? `timeout (${res.timeout}ms)`:null)
            }
          },res.timeout)
        }
      }
    }
    sync = false
    // 判断组件是否解析完成
    return factory.loading ? factory.loadingComp : factory.resolved
  }
}
```

### getFirstComponentChild

```js
// 获取component下第一个子元素
export function getFirstComponentChild(children){
  if(Array.isArray(children)){
    for(let i = 0; i < children.length; i ++){
      const c = children[i]
      if(isDef(c) && (isDef(c.componentOptions) || isAsyncPlaceholder(c))){
        return c
      }
    }
  }
}
```

### isAsyncPlaceholder

```js
export function isAsyncPlaceholder(node){
  return node.isComment && node.asyncFactory
}
```

### normalizeScopedSlots

```js
export function normalizeScopedSlots(slots,normalSlots,prevSlots){
  let res 
  const hasNormalSlots = Object.keys(normalSlots).length > 0
  const isStable = slots ? !!slots.$stable : !hasNoramlSlots
  const key = slots && slots.$key
  // 获取slots
  if(!slots){
    res = {}
  }else if(slots._normalized){
    return slots._normalized
  }else if(isStable && prevSlots && prevSlots !== emptyObject && key === prevSlots.$key && !hasNormalSlots && !prevSlots.$hasNormal){
    return prevSlots
  }else {
    res = {}
    for(const key in slots){
      if(slots[key] && key[0] !== '$'){
        res[key] = normalizeScopeSlot(normalSlots,key,slots[key])
      }
    }
  }
  // 根据normalSlots扩展slots
  for(const key in normalSlots){
    if(!key in res){
      res[key] = proxyNormalSlot(normalSlots,key)
    }
  }
  if(slots && Object.isExtensible(slots)){
    slots._normalized = res
  }
  def(res,'$stable',isStable)
  def(res,'$key',key)
  def(res,'$hasNormal',hasNormalSlots)
  return res
}
```
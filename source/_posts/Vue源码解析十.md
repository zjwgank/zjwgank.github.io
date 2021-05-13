---
title: Vue源码解析十
date: 2021-05-13 22:01:10
tags:
---
```js
function renderMixin(Vue){
  // 向Vue.prototype上挂载属性方法
  installRenderHelpers(Vue.prototype)
  ...
}
```

#### render-static

```js
// 渲染静态DOM
export function renderStatic(index,isInFor){
  // 获取/初始化静态树
  const cached = this._staticTrees || (this._staticTrees = [])
  let tree = cached[index]
  // 如果静态树存在或者没有v-for则直接复用
  if(tree && !isInFor){
    return tree
  }
  // 否则根据渲染方法重新渲染
  tree = cached[index] = this.$options.staticRenderFns[index].call(this._renderProxy,null,this)
  // 并标记key
  markStatic(tree,`__static__${index}`,false)
  return tree
}
```

#### render-list

```js
// 根据val值和render方法生成VNodeList
export function renderList(val,render){
  let ret,i,l,keys,key
  if(Array.isArray(val) || typeof val === 'string'){
    ret = new Array(val.length)
    for(i = 0,l = ret.length; i < l; i ++){
      ret[i] = render(val[i],i)
    }
  }else if(typeof val === 'number'){
    ret = new Array(val)
    for(i = 0; i < val; i ++){
      ret[i] = render(i+1,i)
    }
  }else if(isObject(val)){
    if(hasSymbol && val[Symbol.iterator]){
      ret = []
      const iterator = val[Symbol.iterator]()
      let result = iterator.next()
      while(!result.done){
        ret.push(render(result.value,ret.length))
        result = iterator.next()
      }
    }else{
      keys = Object.keys(val)
      ret = new Array(keys.length)
      for(i = 0,l = keys.length; i < l; i ++){
        key = keys[i]
        ret[i] = render(val[key],key,i)
      }
    }
  }
  if(!isDef(ret)){
    ret = []
  }
  ret._isVList = true
  return ret
}
```

#### render-slot

```js
export function renderSlot(name,fallback,props,bindObject){
  // 获取插槽
  const scopedSlotFn = this.$scopedSlot[name]
  let nodes
  // 根据插槽是否存在是否有传入的属性,生成vnode
  if(scopedSlotFn){
    props = props || {}
    if(bindObject){
      if(process.env.NODE_ENV !== 'production' && !isObject(bindObject)){
        warn('slot v-bind without argument expect an object',this)
      }
      props = extend(extend({},bindObject),props)
      nodes = scopedSlotFn(props) || fallback
    }
  }else{
    nodes = this.$slots[name] || fallback
  }
  const target = props && props.slot
  // 判断插槽是否有插槽,如果存在则根据插槽生成vnode
  if(target){
    return this.$createElement('template',{slot:target},nodes)
  }else{
    return nodes
  }
}
```
#### resolve-filter

```js
// 解析filter
export function resolveFilter(id){
  return resolveAssets(this.$options,'filter',id,true) || identity
}
```

#### resolve-scoped-slots

```js
export function resolveScopedSlots(fns,res,hasDynamicKeys,contentHashKey){
  // 解析传入的插槽
  res = res || { $stable: !hasDynamicKeys }
  for(let i = 0; i < fns.length; i ++){
    const slot = fns[i]
    if(Array.isArray(slot)){
      resolveScopedSlots(slot)
    }else if(slot){
      if(slot.proxy){
        slot.fn.proxy = true
      }
      res[slot.key] = slot.fn
    }
  }
  if(contentHashKey){
    res.$key = contentHashKey
  }
}
```

#### render-slot

```js
export function resolveSlots(children,context){
  if(!children || !children.lenth){
    return {}
  }
  const slots = {}
  for(let i = 0, l = children.length; i < l; i ++){
    const child = children[i]
    const data = child.data
    // 如果child作为Vue的插槽被解析,则移除它的slot属性
    if(data && data.attrs && data.attrs.slot){
      delete data.attrs.slot
    }
    if((child.context === context || child.fnContext === context) && data && data.slot !== null){
      const name = data.slot
      const slot = (slots[name] || (slots[name] = []))
      if(child.tag === 'template'){
        slot.push.apply(slot,child.children || [])
      }else{
        slot.push(child)
      }
    }else{
      (slots.default || (slots.default = [])).push(child)
    }
  }
  for(const name in slots){
    if(slots[name].every(isWhitespace)){
      delet slots[name]
    }
  }
  return slots
}
```

#### check-keycodes

```js
// 反正最后就是返回是否匹配
export function checkKeyCodes(eventKeyCode,key,builtInKeyCode,eventKeyName,builtInKeyName){
  const mappedKeyCode = config.keyCodes[key] || builtInKeyCode
  if(builtKeyName && eventKeyName && !config.keyCodes[key]){
    return isKeyNotMatch(builtInKetName,eventKeyName)
  }else if(mappedKeyCode){
    return isKeyNotMatch(mappedKeyCode,eventKeyCode)
  }else if(eventKeyName){
    return hyphenate(eventKeyName) !== key
  }
}
```

#### bind-object-props

```js
// 绑定属性对象
export function bindObjectProps(data,tag,value,asProp,isSync){
  if(value){
    if(!isObject(value)){
      process.env.NODE_ENV !== 'production' && warn('v-bind without argument expects an Object or Array value',this)
    }else{
      if(Array.isArray(value)){
        value = toObject(value)
      }
      let hash
      for( const key in value){
        if(key === 'class' || key === 'style' || isReserverAttribute(key)){
          hash = data
        }else{
          const type = data.attrs && data.attrs.type
          hash = asProp || config.mustUseProp(tag,type,key) ? data.domProps || (data.domProps = {}) : data.attrs || (data.attrs = {})
        }
        const camelizedKey = camelize(key)
        const hyphenatedKey = hyphenate(key)
        if(!(camelizedKey in hash) && !(hyphenatedKey in hash)){
          hash[key] = value[key]
          // .sync修饰符, $emit(update:key),接收数据更改
          if(isSync){
            const on = data.on || (data.on = {})
            on[`update:${key}`] = function($event){
              value[key] = $event
            }
          }
        }
      }
    }
  }
  return data
}
```

#### bind-object-listeners

```js
export function bindObjectListeners(data,value){
  // 绑定对象监听器
  if(value){
    if(!isPlainObject(value)){
      process.env.NODE_ENV !== 'production' && warn('v-on without argument expects an Object value', this)
    }else{
      const on = data.on = data.on ? extend({},data.on) : {}
      for( const key in value){
        const existing = on[key]
        const ours = value[key]
        on[key] = existing ? [].concat(existing,ours) : ours
      }
    }
  }
  return data
}
```

#### bind-dynamic-keys

```js
// 绑定动态key
export function bindDynamicKeys(baseObj,values){
  for(let i = 0; i < values.length; i += 2){
    const key = values[i]
    if(typeof key === 'string' && key){
      baseObj[values[i]] = values[i + 1]
    }else if(process.env.NODE_ENV !== 'production' && key !== '' && key !== null){
      warn(`Invalid value for dynamic directive argument (expected string or null):${key},`,this)
    }
  }
  return baseObj
}
```
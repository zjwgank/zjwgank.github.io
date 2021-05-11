---
title: Vue源码解析九
date: 2021-05-10 23:04:10
tags:
---
```js
// GlobalAPI
export function initGlobalAPI(Vue){
  // 声明Vue的config读取
  const configDef = {}
  configDef.get = config
  ...
  Object.defineProperty(Vue,'config',configDef)
  // 扩展Vue的公共方法
  Vue.util = {
    warn,extend,mergeOptions,defineReactive
  }
  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick
  Vue.observable = (obj) => {
    observe(obj)
    return obj
  }
  // 声明Vue的组件/指令/过滤
  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })
  Vue.options._base = Vue
  // 扩展组件上的keep-alive指令
  extend(Vue.options.components,builtInComponents)
  // 扩展use方法
  initUse(Vue)
  // 扩展mixin
  initMixin(Vue)
  // 扩展extend
  initExtend(Vue)
  // 初始化Vue的指令/组件/过滤器的配置
  initAssetRegisters(Vue)
}
```

#### builtInComponents

```js
// keep-alive
export default {
  name:'keep-alive',
  abstract:true,
  props:{
    include:[String,RegExp,Array],
    exclude:[String,RegExp,Array],
    max:[String,Number]
  },
  created(){
    this.cache = Object.create(null)
    this.keys = []
  },
  destroyed(){
    // 判断当前dom与缓存是否相同,移除dom并且移除缓存
    for(const ket in this.cache){
      pruneCacheEntry(this.cache,key,this.keys)
    }
  },
  mounted(){
    // 监听include/exclude,移除Cache
    this.$watch('include',val => {
      pruneCache(this,name => matches(val,name))
    })
    this.$watch('exclude',val => {
      pruneCache(this,name => !matches(val,name))
    })
  },
  render(){
    // 获取指令包裹的组件
    const slot = this.$slots.default
    // 获取vnode
    const vnode = getFirstComponentChild(slot)
    // 获取组件配置
    const componentOptions = vnode && vnode.componentOptions
    if(componentOptions){
      const name = getComponentName(componentOptions)
      const { include, exclude } = this
      // 判断过滤组件
      if((include && (!name || !matches(include,name))) || (exclude && name && matches(exclude,name))){
        return vnode
      }
      const { cache, keys } = this
      const key = vnode.key = null ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '') : vnode.key
      if(cache[key]){
        // keep-alive,更新key值
        vnode.componentInstance = cache[key].componentInstance
        remove(keys,key)
        keys.push(key)
      }else{
        // 初始keep-alive,添加key值
        cache[key] = vnode
        keys.push(key)
        if(this.max && keys.length > parseInt(this.max)){
          pruneCacheEntry(cahce,key[0],keys,this._vnode)
        }
      }
      vnode.data.keepAlive = true
    }
    return vnode || (slot && slot[0])
  }
}
```

#### initUse

```js
export function initUse(Vue){
  Vue.use = function(plugin){
    const installedPlugins = (this._installedPlugins || this._installedPlugins = [])
    // 如果插件已安装,直接返回Vue
    if(installedPlugins.indexOf(plugin) > -1){
      return this
    }
    const args = toArray(arguments,1)
    args.unshift(this)
    // 判断插件是否存在install方法
    // 为插件安装传入Vue和参数
    if(typeof plugin.install === 'function'){
      plugin.install.call(plugin,args)
    }else if(typeof plugin === 'function' ){
      plugin.call(null,args)
    }
    installedPlugins.push(plugin)
    return this
  }
}
```

#### initMixin

```js
export function initMixin(Vue){
  Vue.mixin = function(mixin){
    //  扩展options
    this.options = mergeOptions(this.options,mixin)
    return this
  }
}
```

#### initExtend

```js
export function initExtend(Vue){
  Vue.cid = 0
  let cid = 1
  Vue.extend = function(extendOptions){
    extendOptions = extendOptions || {}
    const Super = this
    const SuperId = Super.cid
    const cacheCtors = extendOptions._Ctor|| (extendOptions._Ctor = {})
    if(cacheCtors[SuperId]){
      return cacheCtors[SuperId]
    }
    const name = extendOptions.name || super.options.name
    if(process.env.NODE_ENV !== 'production' && name){
      validComponentName(name)
    }
    // 使用Vue创建一个子类/组件
    const Sub = function VueComponent(options){
      // 根据配置初始化生命周期/数据/props等
      this._init(options)
    }
    // 继承Vue构造器
    Sub.prototype = Object.create(Super.prototype)
    Sub.prototype.constructor = Sub
    Sub.cid = cid ++
    Sub.options = mergeOptions(Super.options,extendOptions)
    Sub['super'] = Super
    // 初始化属性/计算属性
    if(Sub.options.props){
      initProps(Sub)
    }
    if(Sub.options.computed){
      initComputed(Sub)
    }
    // 在子类上挂载父类的方法
    Sub.extend = Super.extend
    Sub.mixin = Super.mixin
    Sub.use = Super.use
    ASSET_TYPES.forEach((type) => {
      Sub[type] = Super[type]
    })
    if(name){
      Sub.options.component[name] = Sub
    }
    Sub.superOptions = Super.options
    Sub.extendOptions = extendOptions
    Sub.sealedOptions = extend({},Sub.options)
    // 缓存子类/组件
    cacheCtors[SuperId] = Sub
    return Sub
  }
}
```

#### initAssetRegisters

```js
export function initAssetRegisters(Vue){
  ASSET_TYPES.forEach(type => {
    Vue[type] = function(id,definition){
      // 获取GlobAPI中缓存的配置
      if(!definition){
        return this.options[type + 's'][id]
      }else{
        // 校验组件名称
        if(process.env.NODE_ENV !== 'production' && type === 'component'){
          validComponentName(id)
        }
        // Vue.extend(definition)创建组件
        if(type === 'component' && isPlainObject(definition)){
          definition.name = definition.name || id
          definition = this.options._base.extend(definition)
        }
        // 返回指令
        if(type === 'directive' && typeof difinition === 'function'){
          definition = { bind: difinition, update: difinition }
        }
        // 缓存配置
        this.options[type + 's'][id] = definition
        return definition 
      }

    }
  })
}
```
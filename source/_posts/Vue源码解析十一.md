---
title: Vue源码解析十一
date: 2021-05-14 22:29:18
tags:
---

### Observer

```js
class Observer{
// 定义属性
  value;
  dep;
  vmCount;
// 构造函数
  constructor(value){
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    // 为value挂载__ob__属性,属性值为实例化ob对象
    def(value,'__ob__',this)
    if(Array.isArray(value)){
      // 判断__proto__是否可用,向value对象上挂载Array属性和方法
      if(hasProto){
        protoAugment(value,arrayMethods)
      }else{
        copyAugment(value,arrayMethods,arrayKeys)
      }
      this.observeArray(value)
    }else{
      this.walk(value)
    }
  }
  // 遍历value进行观察
  walk(obj){
    const keys = Object.keys(obj)
    for(let i = 0;i < keys.length; i ++){
      defineReactive(obj,keys[i])
    }
  }
  observeArray(items){
    for(let i = 0, l = items.length; i < l; i ++){
      observe(items[i])
    }
  }
}
```
#### observe

```js
// 判断当前对象是否被监控,否则实例化
export function observe(value,asRootData){
  if(!isObject(value) || value instanceof VNode){
    return 
  }
  let ob
  if(hasOwn(value,'__ob__') && value.__ob__ instanceof Observer){
    ob = value.__ob__
  }else if(shouldObserve && !isServerRendering() && (Array.isArray(value) || isPlainObject(value)) && Object.isExtensible(value) && !value._isVue){
    ob = new Observer(value)
  }
  if(asRootData && ob){
    ob.vmCount ++
  }
  return ob
}
```

#### defineReactive

```js
export function defineReactive(obj,key,val,customSetter,shallow){
  const dep = new Dep()
  const property = Object.getOwnPropertyDescriptor(obj,key)
  if(property && property.configurable === false){
    return 
  }
  const getter = property && property.get
  const setter = property && property.set
  // 取得默认obj的key属性
  if((!getter || setter) && arguments.length === 2 ){
    val = obj[key]
  }
  // 获取val的Observe
  let childOb = !shallow && observe(val)
  // 重定义key值的读写
  Object.defineProperty(obj,key,{
    enumerable: true,
    configurable: true,
    get: function reactiveGetter(){
      const value = getter ? getter.call(obj) : val
      if(Dep.target){
        dep.depend()
        if(childOb){
          childOb.dep.depend()
          if(Array.isArray(value)){
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter(newVal){
      const value = getter ? getter.call(obj) : val
      if(newVal === value || (newVal !== newVal && value !== value)){
        return
      }
      if(process.env.NODE_ENV !== 'production' && customSetter){
        customSetter()
      }
      if(getter && !setter) return
      if(setter){
        setter.call(obj,newVal)
      }else{
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
  })
}
```

### Dep

```js
let uid = 0
export default class Dep{
  static target;
  id;
  subs;
  constructor(){
    this.id = uid ++
    this.subs = []
  }
  addSub(sub){
    this.subs.push(sub)
  }
  removeSub(sub){
    remove(this.subs,sub)
  }
  depend(){
    if(Dep.target){
      Dep.target.addDep(this)
    }
  }
  notify(){
    const subs = this.subs.slice()
    if(process.env.NODE_ENV !== 'production' && !config.async){
      subs.sort((a,b) => a.id - b.id)
    }
    for(let i = 0, l = subs.length; i < l; i++){
      subs[i].update()
    }
  }
}
Dep.target = null
const targetStatck = []
// 保证Dep.target永远是最新的那个
function pushTarget(target){
  targetStack.push(target)
  Dep.target = target
}
function popTarget(){
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

### Watcher

```js
class Watcher{
  vm,expression,cb,id,deep,user,lazy,sync,dirty,active,deps,newDeps,depIds,newDepIds,before,getter,value;
  
  constructor(vm,expOrFn,cb,options,isRenderWatcher){
    this.vm = vm
    // 挂载到vm的_watchers
    if(isRenderWatcher){
      vm._watcher = this
    }
    vm._watchers.push(this)
    if(options){
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    }else{
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid
    this.active = true
    this.dirty = this.lazy
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production' ? expOrFn.toString() : ''
    if(typeof expOrFn === 'function'){
      this.getter = expOrFn
    }else{
      this.getter = parsePath(expOrFn)
    }
    this.value = this.lazy ? undefined : this.get()
  }

  get(){
    // 设置Dep.target,同时在读取数据时addDep
    pushTarget(this)
    let value
    const vm = this.vm
    try{
      value = this.getter.call(vm,vm)
    }catch(e){
      ...
    }finally{
      if(this.deep){
        traverse(value)
      }
      // 执行完成,清除Dep.target,同时执行cleanupDeps
      popTarget()
      this.cleanupDeps()
    }
    return value
  }

  addDep(dep){
    // 存入dep,dep存入Watcher
    const id = dep.id
    if(!this.newDepIds.has(id)){
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if(!this.depIds.has(id)){
        dep.addSub(this)
      }
    }
  }

  cleanupDeps(){
    // 判断备份dep不存在当前dep,代表永久删除dep当前watcher
    let i = this.deps.length
    while(i --){
      const dep = this.deps[i]
      if(!this.newDepIds.has(dep.id)){
        dep.removeSub(this)
      }
    }
    // 设置depIds和deps作为newDepIds和newDeps的备份
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }

  update(){
    // 判断是否立即执行run-watch回调或者执行updated回调
    if(this.lazy){
      this.dirty = true
    }else if(this.sync){
      this.run()
    }else{
      queueWatcher(this)
    }
  }

  run(){
    // 获取newValue并回调
    if(this.active){
      const value = this.get()
      if(value !== this.value || isObject(value) || this.deep){
        const oldValue = this.value
        this.value = value
        if(this.user){
          try{
            this.cb.call(this.vm,value,oldValue)
          }catch(e){
            handleError(e,this.vm,`callback for watcher "${this.expression}"`)
          }
        }else{
          this.cb.call(this.vm,value,oldValue)
        }
      }
    }
  }

  evalute(){
    this.value = this.get()
    this.dirty = false
  }

  depend(){
    let i = this.deps.length
    while(i --){
      this.deps[i].depend()
    }
  }

  // 清除watcher
  teardown(){
    if(this.active){
      if(!this.vm._isBeingDestroyed){
        remove(this.vm._watchers,this)
      }
      let i = this.deps.length
      while(i --){
        this.deps[i].removeSub(this)
      }
      this.active = false
    }
  }
}
```

### 总结--双向绑定

- defineReactive -- 通过Object.defineProperty重新定义了get和set
  - getter: 判断原属性的get,通过dep添加watcher
  - setter: 判断setter,通过dep调用watcher的update,调用watch回调或者update钩子函数;同时根据newVal设置childOb/Observer,在get时更新
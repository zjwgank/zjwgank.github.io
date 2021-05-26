---
title: Vuex源码解析二
date: 2021-05-25 22:18:32
tags:
---

### install

```js
// Vuex对Vue的依赖
if(!Vue && typeof window !== 'undefined' && window.Vue){
  install(window.Vue)
}
```

#### install

```js
function install(_Vue){
  // 判断是否已经声明Vue,同时是否有重复Use
  if(Vue && _Vue === Vue){
    if(__DEV__){
      console.error(`[vuex] already installed. Vue.use(Vuex) should be called only once`)
    }
    return
  }
  // 赋值Vue对象
  Vue = _Vue
  applyMixin(Vue)
}
```
#### applyMixin

```js
export default function(Vue){
  // 获取引入的vue版本
  const version = Number(Vue.version.split('.')[0])
  // 判断版本,在钩子函数中初始化vuex
  if(version >= 2){
    Vue.mixin({beforeCreate:vuexInit})
  }else{
    // 重写_init
    const _init = Vue.prototype._init
    Vue.prototype._init = function(options = {}){
      options.init = options.init ? [vuexInit].concat(options.init) : vuexInit
      _init.call(this,options)
    }
  }
}
// vuex初始化
function vuexInit(){
  const options = this.$options
  // 注入$store
  if(options.store){
    this.$store = typeof options.store === 'function' ? options.store() : options.store
  }else if(options.parent && options.parent.$store){
    this.$store = options.parent.$store
  }
}
```

### modules

```js
// 设置模块
this._modules = new ModuleCollection(options)
```

#### ModuleCollection

```js
class ModuleCollection{
  constructor(rawRootModule){
    // 注册根节点模块
    this.register([],rawRootModule,false)
  }
  get(path){
    // 根据path从root节点查找子模块
    return path.reduce((module,key) =>{
      return module.getChild(key)
    },this.root)
  }
  getNamespace(path){
    // 获取模块名称
    let module = this.root
    return path.reduce((namespace,key) =>{
      module = module.getChild(key)
      return namespace + (module.namespaced ? key + '/' : '')
    },'')
  }
  update(rawRootModule){
    update([],this.root,rawRootModule)
  }
  register(path,rawModule,runtime = true){
    if(__DEV__){
      // 校验模块的getter/mutation/actionss
      assertRawModule(path,rawModule)
    }
    // 创建新节点
    const newModule = new Module(rawModule,runtime)
    if(path.length === 0){
      // 初始化根节点
      this.root = newModule
    }else{
      // 获取父节点
      const parent = this.get(path.lice(0,-1))
      // 为父节点添加对应key值的模块
      parent.addChild(path[path.length - 1],newModule)
    }
    if(rawModule.modules){
      // 遍历根节点下子模块注册
      forEachValue(rawModule.modules,(rawChildModule,key) => {
        // 注册子模块
        this.register(path.concat[key],rawChildModule,runtime)
      })
    }
  }
  unregister(path){
    // 获取父节点
    const parent = this.get(path.slice(0,-1))
    const key = path[path.length - 1]
    // 获取要删除的子节点
    const child = parent.getChild(key)
    if(!child){
      if(__DEV__){
        console.warn(`[vuex] tring to unregister module '${key}', which is not registerd`)
      }
      return
    }
    // 非运行时
    if(!child.runtime){
      return
    }
    // 删除子模块
    parent.removeChild(key)
  }
  isRegistered(path){
    // 获取父节点
    const parent = this.get(path.slice(0,-1))
    const key = path[path.length - 1]
    // 判断子节点是否存在
    if(parent){
      return parent.hasChild(key)
    }
    return false
  }
}
```

##### assertRawModule

```js
const functionAssert = {
  assert: value => typeof value === 'function', // 标注getter/mutations的类型
  expected:'function'
}
const objectAssert = {
  assert:value => typeof value === 'function' || (typeof value === 'object' && typeof value.handler === 'function'),
  expected:'function or object with "handler" function'
}
// 标注modules中的函数类型
const assertTypes = {
  getters:functionAssert,
  mutations:functionAssert,
  actions:objectAssert
}
function assertRawModule(path,rawModule){
  Object.keys(assertTypes).forEach(key => {
    // 判断模块缺少getter/actions/mutations
    if(!rawModule[key]) return
    const assertOptions = assertTypes[key]
    // 遍历方法
    forEachValue(rawModule[key],(value,type) => {
      // 判断方法类型是否符合
      assert(assertOptions.assert(value),makeAssertionMessage(path,key,type,value,assertOptions.expected))
    })
  })
}
// 根据模块名输出方法字段是否符合规范
function makeAssertionMessage(path,key,type,value,assertOptions.expected){
  let buf = `${key} should be ${expected} but "${key}.${type}"`
  if(path.length > 0){
    buf += `in module "${path.join('.')}"`
  }
  buf += ` is ${JSON.stringify(value)}`
  return buf
}
```

##### update

```js
function update(path,targetModule,newMoudle){
  // 校验newModule的字段方法
  if(__DEV__){
    assertRawMoudle(path,newModule)
  }
  // 更新根节点
  targetMoudle.update(newModule)
  if(newModule.modules){
    for(const key in newModule.modules){
      if(!targetModule.getChild(key)){
        if(__DEV__){
          console.warn(`[vuex] trying to add a new module '${key}' on hot reloading, manual reload is needed`)
        }
        return 
      }
      // 更新子模块
      update(path.concat(key),targetModule.getChild(key),newModule.modules[key])
    }
  }
}
```

#### Module

```js
class Module{
  constructor(rawModule,runtime){
    // 获取运行时
    this.runtime = runtime
    // 子模块
    this._children = Object.create(null)
    this._rawModule = rawModule
    // 获取模块的state
    const rawState = rawModule.state
    this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
  }
  get namespaced(){
    return !!this._rawModule.namespaced
  }
  // 为模块添加子节点
  addChild(key,module){
    this._children[key] = module
  }
  // 移除子节点
  removeChild(key){
    delete this._children[key]
  }
  // 获取子节点
  getChild(key){
    return this._children[key]
  }
  // 判断是否包含子节点
  hasChild(key){
    return key in this._children
  }
  update(rawModule){
    // 更新当前模块
    this._rawModule.namspaced = rawModule.namespaced
    if(rawModule.actions){
      this._rawModule.actions = rawModule.actions
    }
    if(rawModule.mutations){
      this._rawModule.mutations = rawModule.mutations
    }
    if(rawModule.getters){
      this._rawModule.getters = rawModule.getters
    }
  }
  // 遍历子模块,执行fn
  forEachChild(fn){
    forEachValue(this._children,fn)
  }
  // 遍历getters
  forEachGetter(fn){
    if(this._rawModule.getters){
      forEachValue(this._rawModule.getters,fn)
    }
  }
  // 遍历actions
  forEachAction(fn){
    if(this._rawModule.actions){
      forEachValue(this._rawModule.actions,fn)
    }
  }
  // 遍历mutations
  forEachMutation(fn){
    if(this._rawModule.mutations){
      forEachValue(this._rawMoudle.mutations,fn)
    }
  }
}
```

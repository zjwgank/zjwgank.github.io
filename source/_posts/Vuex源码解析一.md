---
title: Vuex源码解析一
date: 2021-05-21 21:22:21
tags:
---

### Vuex

#### 打包

```js
// package.json
{
  name:'vuex',
  ...
  main:'dist/vuex.common.js',
  module:'dist/vuex.esm.js'
  ...
}
// rollup.main.config.js
export default createEntries([
  ...
  {input:'src/index.js',file:'dist/vuex.esm.js',format:'es',env:'development'},
  ...
  {input:'src/index.cjs.js',file:'dist/vuex.common.js',format:'cjs',ebv:'development'}
])
```

#### vuex

```js
import { Store, install } from './store'
import { mapState, mapMutations, mapGetters, mapActions, createNamespaceHelpers } from './helpers'
import createLogger from './plugins/logger'

export default{
  Store,
  install,
  version:'__VERSION__',
  mapState,
  mapMutations,
  mapGetters,
  mapActions,
  createNamespaceHelpers,
  createLogger
}
```

#### Store

```js
// 判断是否引入Vue
let Vue
class Store{
  constructor(options = {}){
    // 判断当前Vue不存在,同时处于浏览器下,Vue已挂载,为Vue变量赋值
    if(!Vue && typeof window !== 'undefined' && window.Vue){
      install(window.Vue)
    }
    const { plugins = [], strict = false } = options 
    // 赋值私有变量
    this._commiting = false
    this._actions = Object.create(null)
    this._actionSubscribers = []
    this._mutations = Object.create(null)
    this._wrappedGetters = Object.create(null)
    this._modules = new ModuleCollection(options)
    this._modulesNamespaceMap = Object.create(null)
    this._subscribers = []
    this._watcherVM = new Vue()
    this._makeLocalGettersCache = Object.create(null)
    // 将dispatch和commit的上下文指向绑定
    const store = this
    const { dispatch, commit } = this
    this.dispatch = function boundDispatch(type,payload){
      return dispatch.call(store,type,payload)
    }
    this.commit = function boundCommit(type,payload,options){
      return commit.call(store,type,payload,options)
    }
    this.strict = strict
    const state = this._modules.root.state
    // 绑定全局action/getter/mutationss
    installModule(this,state,[],this._modules.root)
    // 初始化store绑定的vm对象
    resetStoreVM(this,state)
    // 应用plugin
    plugins.forEach(plugin => plugin(this))
    const useDevtools = options.devtools !== undefined ? options.devtools : Vue.config.devtools
    if(useDevtools){
      devtoolPlugin(this)
    }
  }
  get state(){
    return this._vm._data.$$state
  }
  set state(v){
    if(__DEV__){
      asset(false,`use store.replaceState() to explicit replace store state`)
    }
  }
  commit(_type,_payload,_options){
    // 格式化获取type,payload,options
    const { type, payload, options} = unifyObjectStyle(_type,_payload,_options)
    const mutation = { type, payload }
    // 获取同步事件
    const entry = this._mutations[type]
    if(!entry){
      if(__DEV__){
        console.error(`[vuex] unkonwn mutation type:${type}`)
      }
      return
    }
    // 执行同步事件逻辑
    this._withCommit(() => {
      entry.forEach(function commitIterator(handler){
        handler(payload)
      })
    })
    this._subscribers.slice().forEach(sub => sub(mutation,this.state))
    if(__DEV__ && options && options.slient){
      console.warn(`[vuex] mutation type: ${type}. Slient option has been removed. use the filter funtionality in the vue-devtools`)
    }
  }
  dispatch(_type,_payload){
    // 获取type/paylod
    const { type, payload }= unifyObjectStyle(_type,_payload)
    const action = { type, payload }
    const entry = this.actions[type]
    if(!entry){
      if(__DEV__){
        console.error(`[vuex] unkonwn action type:${type}`)
      }
      return
    }
    // 执行插件的before操作
    try{
      this._actionSubscribers.slice().filter(sub => sub.before).forEach(sub => sub.before(action,this.state))
    }catch(e){
      ...
    }
    // 根据dispatch的事件获取执行结果
    const result = entry.length > 1 ? Promise.all(entry.map(handler => handler(payload))) : entry[0].payload
    return new Promise((resolve,reject) => {
      result.then(res => {
        // 执行plugin.after
        try{
          this._actionSubscribers.slice().filter(sub => sub.after).forEach(sub => sub.afetr(action,this.state))
        }catch(e){
          ...
        }
        resolve(res)
      })
    },error => {
      // plugin.error
      try{
        this._actionSubscribers.slice().filter(sub => sub.error).forEach(sub=>sub.error(action,this.state,error))
      }catch(e){
        ...
      }
      reject(error)
    })
  }
  subscribe(fn,options){
    // 订阅commit事件,同时返回一个函数清除这个事件
    return genericSubscribe(fn,this._subscribers,options)
  }
  subscribeAction(fn,options){
    // 订阅action事件,同时返回一个函数清除这个事件
    const subs = typeof fn === 'function' ? { before: fn} : fn
    return genericSubscribe(fn,this._actionSubscribers,options)
  }
  watch(getter,cb,options){
    if(__DEV__){
      asset(typeof getter === 'function',`store.watch only accepts a function`)
    }
    // 响应式的监听
    return this._watcherVM.$watch(() => getter(this.state,this.getters),cb,options)
  }
  replaceState(state){
    // 替换state
    this._withCommit(() => {
      this._vm._data.$$state = state
    })
  }
  registerModule(path,rawModule,options = {}){
    if(typeof path === 'string') path = [path]
    if(__DEV__){
      assert(Array.isArray(path),`module path must be a string or a Array`)
      assert(path.length > 0,`cannot register the root module by using registerModule`)
    }
    // 动态注册模块
    this._modules.register(path,rawModule)
    // 动态注册模块的事件/方法/state
    installModule(this,this.state,path,this._module.get(path),options.prevState)
    resetStoreVM(this,this.state)
  }
  unregisterModule(path){
    if(typeof path === 'string') path = [path]
    if(__DEV__){
      asset(Array.isArray(path),`module path must be a string or a array`)
    }
    // 卸载模块
    this._modules.unregister(path)
    // 移除state
    this._withCommit(() => {
      const parentState = getNestedState(this.state,path.slice(0,-1))
      Vue.delete(parentState,path[path.length - 1])
    })
    resetStore(this)
  }
  hasModule(path){
    if(typeof path === 'string') path = [path]
    if(__DEV__){
      asset(Array.isArray(path),`module path must be a string or a array`)
    }
    // 检测模块是否已被注册
    return this._modules.isRegistered(path)
  }
  hotUpdate(newOptions){
    // 热替换更新module/actions
    this._modules.update(newOptions)
    // 更新根节点状态
    resetStore(this)
  }
  _withCommit(fn){
    // 更新状态,执行事件
    const commiting = this._committing
    this._committing = true
    fn()
    this._committing = committing
  }
}
```
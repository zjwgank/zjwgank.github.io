---
title: Vuex源码解析三
date: 2021-05-26 23:13:49
tags:
---

### _watcherVM

```js
this._watcherVM = new Vue()
// 监听watcher
Vue.prototype.$watch = function(expOrFn,cb,options){
  var vm = this
  if(isPlainObject(cb)){
    // 根据cb创建监听
    return createWatcher(vm,expOrFn,cb,options)
  }
  options = options || {}
  options.user = true
  // 创建监听器
  var watcher = new Watcher(vm,expOrFn,cb,options)
  // 立即执行返回
  if(options.immediate){
    try{
      cb.call(vm,watcher.value)
    }catch(e){
      handleError(error,vm,('callback for immediate watcher' + watcher.expression))
    }
  }
  // 移除watcher
  return function unwatchFn(){
    watcher.teardown()
  }
}
```

### installModule

```js
const state = this._modules.root.state
installModule(this,state,[],this._modules.root)
```

#### installModule

```js
function installModule(store,rootState,path,module,hot){
  // 判断是否处于根节点
  const isRoot = !path.length
  // 获取根节点的模块名
  const namespace = store._modules.getNamespace(path)
  // 子模块
  if(module.namespaced){
    // 判断子模块已经存在
    if(store._modulesNamespaceMap[namespace] && __DEV__){
      console.error(`[vuex] duplicate namespace ${namespace} for the namespaced module ${path.join('/')}` )
    }
    // 设置modulesNamespaceMap
    store._modulesNamespaceMap[namespace] = module
  }
  if(!isRoot && !ishot){
    // 遍历获取父节点的State
    const parentState = getNestedState(rootState,path.slice(0,-1))
    const moduleName = path[path.length - 1]
    store._withCommit(() =>{
      if(__DEV__){
        // 模块字段在父节点state
        if(moduleName in parentState){
          console.warn(`[vuex] state field "${moduleName}" was overriden by a module with the same name at "${path.join(".")}"`)
        }
      }
      // 在父节点设置moduleName的state
      Vue.set(parentState,moduleName,module.state)
    })
  }
  // 设置模块的上下文方法
  const local = module.context = makeLocalContext(store,namespace,path)
  // 注册mutation
  module.forEachMutation((mutation,key) =>{
    const namespacedType = namespace + key
    registerMutation(store,namespacedType,mutation,local)
  })
  // 注册action
  module.forEachAction((action,key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store,type,handler,local)
  })
  // 注册getter
  module.forEachGetter((getter,key) =>{
    const namespacedType = namespace + key
    registerGetter(store,namespacedType,getter,local)
  })
  // 注册子模块
  module.forEachChild((child,key) =>{
    installModule(store,rootState,path.concat(key),child,hot)
  })
}
```

##### getNestedState

```js
function getNestedState(state,path){
  // reduce依此通过state查找
  return path.reduce((state,key) => state[key],state)
}
```

##### makeLocalContext

```js
function makeLocalContext(store,namespace,path){
  // 根节点
  const noNamespace = namespace === ''
  const local = {
    // 根节点采用dispatch否则重载
    dispatch: noNamespace ? store.dispatch : (_type,_payload,_options) =>{
      const args = unifyObjectStyle(_type,_payload,_options)
      const { payload, options } = args
      let { type } = args
      // 设置子模块事件名称
      if(!options || !options.root){
        type = namespace + type
        // 查找不到action
        if(__DEV__ && !store._actions[type]){
          console.warn(`[vuex] unknown local action type: ${args.type},global type:${type}`)
          return
        }
      }
      return store.dispatch(type,payload)
    },
    commit:noNamespace ? store.commit : (_type,_payload,_options) =>{
      const args = unifyObjectStyle(_type,_payload,_options)
      const { payload, options } = args
      let { type } = args
      // 设置子模块mutatiaon
      if(!options || !options.hot){
        type = namespace + type
        if(__DEV__ && !store._mutations[type]){
          console.error(`[vuex] unknown local mutation type:${args.type}, global type:${type}`)
          return 
        }
      }
      return store.commit(type,payload)
    }
  }
  Object.defineProperties(local,{
    // getters获取
    getters:{
      get: noNamespace ? () => store.getters : () => makeLocalGetters(store,namespace)
    },
    // state获取
    state:{
      get: () => getNestedState(store.state,path)
    }
  })
  return local
}
```

##### registerMutation

```js
function registerMutation(store,type,handler,local){
  // 注册mutation, type =  模块 + 方法
  const entry = store._mutations[type] || (store._mutations[type] = [])
  // _mutations添加事件
  entry.push(function wrappedMutationHandler(payload){
    handler.call(store,local.state,payload)
  })
}
```

##### registerAction

```js
function registerAction(store,type,handler,local){
  // 获取actions
  const entry = store._actions[type] || (store._actions[type] = [])
  entry.push(function wrappedActionHandler(payload){
    // 设置action
    let res = handler.call(store,{
      dispatch:local.dispatch,
      commit:local.commit,
      getters:local.getters,
      state:local.state,
      rootGetters:store.getters,
      rootState:store.state
    },payload)
    if(!isPromise(res)){
      res = Promise.resolve(res)
    }
    if(store._devtoolHook){
      return res.catch(err => {
        store._devtoolHook.emit('vuex:error',err)
        throw err
      })
    }else{
      return res
    }
  })
}
```

##### registerGetter

```js
function registerGetter(store,type,rawGetter,local){
  if(store._wrappedGetters[type]){
    if(__DEV__){
      console.error(`[vuex] duplicate getter key:${type}`)
      return 
    }
  }
  // 注册getters
  store._wrapperGetters[type] = function wrapperGetter(store){
    return rawGetter(local.state,local.getters,store.state,store.getters)
  }
}
```

##### makeLocalGetters

```js
function makeLocalGetters(store,namespace){
  if(!store._makeLocalGetterCache[namespace]){
    const gettersProxy = {}
    const splitPos = namespace.length
    // 遍历getter设置给getterProxy 挂载到namespaceCache
    Object.keys(store.getters).forEach(type => {
      if(type.slice(0,splitPos) !== namespace) return
      const localType = type.slice(0,splitPos)
      Object.defineProperty(gettersProxy,localType,{
        get:() => store.getters[type],
        enumerable:true
      })
    })
    store._makeLocalGetterCache[namespace] = gettersProxy
  }
  return store._makeLocalGetterCache[namespace]
}
```
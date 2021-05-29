---
title: Vuex源码解析四
date: 2021-05-28 22:42:18
tags:
---
### resetStoreVM

```js
function resetStoreVM(store,state,hot){
  const oldVm = store._vm
  store.getters = {}
  store._makeLocalGettersCache = Object.create(null)
  // 获取getters
  const wrappedGetters = store._wrapperGetters
  const computed = {}
  forEachValue(wrappedGetters,(fn,key) => {
    // 遍历getter
    computed[key] = partial(fn,store)
    // 定义getter的获取
    Object.defineProperty(store.getters,key,{
      get:() => store._vm[key],
      enumreable:true
    })
  })
  const silent = Vue.config.silent
  Vue.config.silent = true
  // 将state和computed属性配置到vm
  store._vm = new Vue({
    data:{
      $$state:state
    },
    computed
  })
  Vue.config.silent = silent
  // 严格模式,不能在mutation之外操控state
  if(store.strict){
    enableStrictMode(store)
  }
  // 销毁旧数据和vm
  if(oldVm){
    if(hot){
      store._withCommit(() => {
        oldVm._data.$$satte = null
      })
      Vue.nextTiclk(() => oldVm.$destroy())
    }
  }
}
```

#### enableStrictMode

```js
function enableStrictMode(store){
  // 监听state, 非commit不能操作state
  store._vm.$watch(function(){ return this._data.$$state},() => {
    if(__DEV__){
      assert(store._committing,`do not mutate vuex store state outside mutation handler`)
    }
  },{ deep:true, aync: true})
}
```

#### resetStore

```js
function resetStore(store,hot){
  // 重置actions/mutations/getters
  store._actions = Object.create(null)
  store._mutations = Object.create(null)
  store._wrappedGetters = Object.create(null)
  store._modulesNamespaceMap = Object.create(null)
  const state = store.state
  // 重新注册模块action等
  installModule(store,state,[],store._modules.root,true)
  // 重置state/getter
  resetStoreVM(store)
}
```

### createNamespacedHelpers

```js
// 基于命名空间的辅助函数
const createNamespacedHelpers = (namespace) => ({
  mapState: mapState.bind(null,namespace),
  mapGetters:mapGetters.bind(null,namespace),
  mapMutations: mapMutations.bind(null,namespace),
  mapActions: mapActions.bind(null,namespace)
})
```
#### normalizeNamespace

```js
function normalizeNamespace(fn){
  // 通过闭包返回一个函数,调用fn
  return (namespace,map) =>{
    // 判断命名空间是否是字符串且最后一个字符'/'
    if(typeof namespace !== 'string'){
      map = namespace 
      namespace = ''
    }else if(namespace.charAt(namespace.length - 1) !== '/'){
      namespace += '/'
    }
    return fn(namespace,map)
  }
}
```

#### normalizeMap

```js
function normalizeMap(map){
  // map非数组或对象返回[]
  if(!isValidMap(map)){
    return []
  }
  // 转换成对象
  return Array.isArray(map) ? map.map(key => ({key,val:key})) : Object.keys(map).map(key => ({key,val:map[key]}))
}
```

### mapState

```js
export const mapState = normalizeNamespace((namespace,states) => {
  const res = {}
  // 校验state是个数组或者对象
  if(__DEV__ && !isValidMap(states)){
    console.error(`[vuex] mapState: mapper parameter must be either an Array or an Object`)
  }
  normalizeMap(states).forEach(({key,val}) => {
    // 返回字段的对应获取方法
    res[key] = function mappedState(){
      // 获取state/getter(根节点或者子节点)
      let state = this.$store.state
      let getters = this.$store.getters
      if(namespace){
        const module = getModuleByNamespace(this.$store,'mapState',namespace)
        if(!module){
          return
        }
        state = module.context.state
        getters = module.context.getters
      }
      return typeof val === 'function' ? val.call(this,state,getters) : state[val]
    }
    // 声明它是vuex属性
    res[key].vuex = true
  })
  return res
})
```

### mapMutations

```js
export const mapMutations = normalizeNamespace((namespace,mutations) => {
  const res = {}
  if(__DEV__ && !isValidMap(mutations)){
    console.error(`[vuex] mapMutations: mapper parameter must be either an Array or An Object`)
  }
  normalizeMap(mutations).forEach(({key,val}) =>{
    // 为vue设置key对象的方法
    res[key] = function mappedMutation(...args){
      // 获取模块上的commit函数
      let commit = this.$store.commit
      if(namespace){
        const module = getModuleByNamespace(this.$store,'mapMutations',namespace)
        if(!module){
          return
        }
        commit = module.context.commit
      }
      return typeof val === 'function' ? val.apply(this,[commit].concat(args)) : commit.apply(this.$store,[val].concat(args))
    }
  })
  return res
})
```

### mapGetters

```js
export const mapGetters = normalizeNamespce((namespace,getters) =>{
  const res = {}
  if(__DEV__ && !isValidMap(getters)){
    console.error(`[vuex] mapGetters: mapper parameter must be either an Array or an Object`)
  }
  normalizeMap(getters).forEach(({key,val}) => {
    // getterName 获取
    val = namespace + val
    res[key] = function mapGetters(){
      // module不存在
      if(namespace && !getModuleByNamespace(this.$store,'mapGetters',namespace)){
        return
      }
      // 判断getter不存在
      if(__DEV__ && !(val in this.$store.getters)){
        console.error(`[vuex] unknown getter:${val}`)
      }
      return this.$store.getters[val]
    }
    res[key].vuex = true
  })
  return res
})
```

### mapActions

```js
export const mapActions = normalizeNamespace((namespce,actions) =>{
  const res = {}
  if(__DEV__ && !isValidMap(actions)){
    console.error(`[vuex] mapActions: mapper parameter must be either an Array or an Object`)
  }
  normalizeMap(actions).forEach(({key,val}) => {
    res[key] = function mappedActions(...args){
      // 获取对应模块的dispatch
      let dispatch = this.$store.dispatch
      if(namespace){
        const module = getModuleByNamespace(this.$store,'mapActions',namespace)
        if(!module){
          return
        }
        dispatch = module.context.dispatch
      }
      // 返回dispatch的执行
      return typeof val === 'function' ? val.apply(this,[dispatch].concat(args)) : dispatch.apply(this.$store,[val].concat(args))
    }
  })
  return res
})
```
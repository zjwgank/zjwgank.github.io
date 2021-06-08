---
title: VueRouter源码解析二
date: 2021-06-03 22:42:50
tags:
---

```js
VueRouter.install = install
VueRouter.version = '__VERSION__'
VueRouter.isNavigationFailure = isNavigationFailure
VueRouter.NavigationFailureType = NavigationFailureType
VueRouter.START_LOCATION = START
```

### install

```js
export let _Vue
export function install(Vue){
  // VueRouter已经引入
  if(install.installed && _Vue === Vue) return
  // 设置VueRouter已引入
  install.installed = true
  _Vue = Vue
  const isDef = v => v !== undefined
  // 注册路由实例
  const registerInstance = (vm,callVal) => {
    // 获取父节点
    let i = vm.$options._parentVnode
    // 注册路由实例
    if(isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouterInstance)){
      i(vm,callVal)
    }
  }
  Vue.mixin({
    // 添加beforeCreate钩子函数
    beforeCreate(){
      // 判断router配置
      if(isDef(this.$options.router)){
        this._routerRoot = this
        this._router = this.$options.router
        // 初始化router
        this._router.init(this)
        // 定义_route的获取
        Vue.util.defineReactive(this,'_route',this._router.history.current)
      }else{
        // 设置子节点的routerRoot
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      // 注册router实例
      registerInstance(this,this)
    },
    // destroyed钩子函数
    destroyed(){
      // 销毁路由实例
      registerInstance(this)
    }
  })
  // 定义$router属性获取
  Object.defineProperty(Vue.prototype,'$router',{
    get(){ return this._routerRoot._router}
  })
  // 定义$route获取
  Object.defineProperty(Vue.prototype,'$route',{
    get() { return this._routerRoot._route}
  })
  // 添加RouterView/RouterLink组件
  Vue.component('RouterView',view)
  Vue.component('RouterLink',link)
  const strats = Vue.config.optionMergeStrategies
  // 使用created函数来操作router钩子
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}
```

#### RouterView

```js
// 函数式组件
export default {
  name: 'RouterView',
  functional:true,
  props:{
    name:{type:String,default:'default'},
  },
  render(_,{props,children,parent,data}){
    // 标记作为路由界面
    data.routerView = true
    // 采用父组件的createElement
    const h = parent.$createElement
    // 路由名称
    const name = props.name
    // 父组件的路由
    const router = parent.$route
    // 缓存路由组件
    const cache = parent._routerViewCache || (parent._routerViewCache = {})
    // 记录当前路由深度
    let depth = 0
    let inactive = false
    // 记录路由深度/渲染keepAlive的缓存界面
    // 判断父组件非根节点
    while(parent && parent._routerRoot !== parent){
      // 获取父组件的虚拟dom
      const vnodeData = parent.$vnode ? parent.$vnode.data : {}
      // 如果组件为路由组件，路由深度自增
      if(vnodeData.routerView){
        depth ++
      }
      // keepAlive，记录是否使用缓存节点
      if(vnodeData.keepAlive && parent._directInactive && parent._inactive){
        inactive  = true
      }
      // 向根节点递归
      parent = parent.$parent
    }
    // 记录当前节点的路由深度
    data.routerViewDepth = depth
    // keep-alive
    if(inactive){
      // 使用缓存数据和组件
      const cacheData = cache[name]
      const cacheComponent = cacheData && cacheData.component
      // 根据组件是否存在进行渲染
      if(cacheComponent){
        // 判断路由组件是否传入参数,参数类型要求boolean/object/function
        if(cacheData.configProps){
          // 将路由组件的参数克隆到组件的属性上
          fillPropsinData(cacheComponent,data,cacheData.route,cacheData.configProps)
        }
        // 渲染组件
        return h(cacheComponent,data,children)
      }else{
        return h()
      }
    }
    // 如果组件非keepAlive,通过钩子函数组册组件实例，缓存组件实例/属性/路由
    // 从路由深度查找匹配的组件
    const matched = route.matched[depth]
    const component = matched && matched.components[name]
    if(!matched || !component){
      cache[name] = null
      return h()
    }
    cache[name] = { component }
    data.registerRouteInstance = (vm,val) => {
      const current = matched.instances[name]
      // beforeCreate未注册实例进行注册， destroy 销毁实例
      if((val && current !== vm) || (!val && current === vm)){
        matched.instances[name] = vm
      }
    }
    // prepatch钩子注册组件实例用作复用
    ;(data.hook || (data.hook = {})).prepatch = (_,vnode) => {
      matched.instances[name] = vnode.componentInstance
    }
    data.hook.init = (vnode) => {
      // keepAlive组件在init时注册组件实例
      if(vnode.data.keepAlive && vnode.componentInstance && vnode.componentInstance !== matched.instances[name]){
        matched.instances[name] = vnode.componentInstance
      }
      // 匹配路由实例，调用路由开始时的回调
      handleRouterEntered(route)
    }
    const configProps = matched.props && matched.props[name]
    // 缓存路由和路由属性
    if(configPorps){
      extend(cache[name],{
        route,
        configProps
      })
      fillPropsinData(component,data,route,configProps)
    }
    // 渲染路由界面
    return h(component,data,children)
  }
}
```

#### RouterLink

```js
let warnedCustomSlot
let warnedTagProp
let warnedEventProp
export default{
  name:'RouterLink',
  props:{
    to:{ type: toTypes, required: true}, // 路由跳转
    tag: { type:String, default: 'a'}, // 路由模拟的组件 
    custom:Boolean, // 
    exact:Boolean, // 是否精准匹配路由
    exactPath:Boolean, // 精准匹配的路由路径
    append:Boolean, // 是否在当前路由上追加
    replace:Boolean, // routerReplace 替换 routerPush
    activeClass:String, // 匹配到时class
    exactActiveClass:String, // 精准匹配时class
    ariaCurrentValue:{ type: String, default: 'page'}, // 链接精准匹配时的配置的aria-current
    event: {type: eventTypes, default:'click'} // 触发导航时的事件
  },
  render(h){
    // 获取路由模块
    const router = this.$router
    // 获取当前路由
    const current = this.$route
    // 解析目的链接
    const { location, route, href } = router.resolve(this.to,current,this.append)
    const classes = {}
    // 标记全局路由配置的匹配class和精准匹配class
    const globalActiveClass = router.options.linkActiveClass
    const globalExactActiveClass = router.options.linkExactActiveClass
    // 设置默认class
    const activeClassFallBack = globalActiveClass == null ? 'router-link-active' : globalActiveClass
    const exactActiveClassFallBack = globalExactActiveClass == null ? 'router-link-exact-active' : globalExactActiveClass
    // 设置当前路由的匹配class
    const activeClass = this.activeClass == null ? activeClassFallBack : this.activeClass
    const exactActiveClass = this.exactActiveClass == null ? exactActiveClassFallBack : this.exactActiveClass
    // 重定向进行路由对象创建或者采用原路由对象
    const compareTarget = route.redirectedFrom ? createRoute(null,normalizeLocation(route.redirectedFrom),null,router) : route
    // 添加当前路由的class状态
    classes[exactActiveClass] = isSameRoute(current,compareTarget,this.exactPath)
    classes[activeClass] = this.exact || this.exactPath ? classes[exactiveClass] : isIncludedRoute(current,compareTarget)
    const ariaCurrentValue = classes[exactActiveClass] ? this.ariaCurrentValue : null
    // 判断路由守卫/进行路由跳转
    const handler = e => {
      // 路由守卫判断键盘控制/代理/右键/_blank
      if(guardEvent(e)){
        if(this.replace){
          router.replace(location,noop)
        }else{
          router.push(location,noop)
        }
      }
    }
    // 注册link的导航触发事件
    const on = {click:guardEvent}
    if(Array.isArray(this.event)){
      this.event.forEach(e => {
        on[e] = handler
      })
    }else{
      on[this.event] = handler
    }
    const data = {class:classes}
    // 渲染v-slot的link组件
    const scopedSlot = !this.$scopedSlots.$hasNormal && this.$scopedSlots.default && this.$scopedSlots.default({href,route,navigate:handler,isActive:classes[activeClass],isExactActive:classes[exactActiveClass]})
    if(scopedSlot){
      ...// custom属性用于 v-slot
      // v-slot 没有to属性
      if(scopedSlot.length === 1){
        return scopedSlot[0]
      }else if(scopedSlot.length > 1 || !scopedSlot.length){
        ...// 提示v-slot && to的渲染 
        // 不存在v-slot和to返回空元素，同时存在则使用span标签包裹
        return scopedSlot.length === 0 ? h() : h('span',{},scopedSlot)
      }
    }
    if(process.env.NODE_ENV !== 'production'){
      ...// VueRouter 4版本 要求使用v-slot代替tag属性
      ...// VurRouter 4版本 要求使用v-slot代替event属性
    }
    if(this.tag === 'a'){
      data.on = on
      data.attrs = { href, 'aria-current':ariaCurrentValue }
    }else{
      // 查询子节点的锚点a标签
      const a = findAnchor(this.$slots.default)
      if(a){
        // 存在a标签，整合属性和事件
        a.isStatic = false
        const aData = (a.data = extend({},a.data))
        aData.on = aData.on || {}
        // 整合a标签和link上配置的事件
        for(const event in aData.on){
          const handler = aData.on[event]
          if(event in on){
            aData.on[event] = Array.isArray(handler) ? handler : [handler]
          }
        }
        for(const event in on){
          if(event in aData.on){
            aData.on[event].push(on[event])
          }else{
            aData.on[event] = [on[event]]
          }
        }
        // 设置属性
        const aAttrs = (a.data.attrs = extend({},a.data.attrs))
        aAttrs.href = href
        aAttrs['aria-current'] = arriaCurrentValue
      }else{
        data.on = on
      }
    }
    // 渲染link
    return h(this.tag,data,this.$slot.default)
  }
}
```
### isNavigationFailure

```js
// 校验是否routerError：redirect/abort/cancel/duplicate
export function isNavigationFailure(err,errorType){
  return isError(err) && err._isRouter && (errorType == null || err.type === errorType)
}
```

### NavigationFailureType

```js
// 各种route报错类型
export const NavigationFailureType = {
  redirected:2,
  aborted:4,
  canceled:8,
  duplicated:16
}
```
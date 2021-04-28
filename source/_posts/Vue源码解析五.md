---
title: Vue源码解析五
date: 2021-04-28 20:19:54
tags:
---
```js
// 替换Vue的Components方法中的Transition
extend(Vue.options.components,platformComponents)
```

#### transition

```js
export default{
  name:'transition',
  props:transitionProps,// 过渡状态相关属性
  abstract:true,
  render(h){
    // 获取过渡状态应用的dom
    let children = this.$slots.default
    // 如果应用的dom不存在或者为textNode,则不做操作
    if(!children){
      return
    }
    children = children.filter(isNotTextNode)
    if(!children.length){
      return 
    }
    // 多个dom状态改变采用transitionGroup,mode是否对应Vue的提供
    ...
    const rawChild = children[0]
    // 当前dom的父元素存在transition时 跳过transition
    if(hasParentTransition(rawChild)){
      return rawChild
    }
    // 当前dom的子节点存在keep-alive时 跳过transition
    const child = getRealChild(rawChild)
    if(!child){
      return rawChild
    }
    if(this._leaving){
      // 当前元素先进行过渡
      return placeholder(h,rawChild)
    }
    ...
    const data = (child.data || (child.data = {})).transition = extractTransitionData(this)
    const oldRawChild = this._vnode
    const oldChild = getRealChild(oldRawChild)
    // 判断v-show指令带来的过渡状态改变
    if(child.data.directives && child.data.directives.some(isVShowDirective)){
      child.data.show = true
    }
    // 比对前后节点变化
    if(oldChild && oldChild.data && !isSameChild(child,oldChild) && !isAsyncPlaceholder(oldChild) && !(oldChild.componentInstance && oldChild.componentInstance._vnode.isComment)){
      const oldData = oldChid.data.transition = extend({},data)
      if(mode === 'out-in'){
        // 当前元素先进性过渡,完成后新元素过渡进入
        this._leaving = true
        mergeVNodeHook(oldData,'afterLeave',() => {
          this._leaving = false;
          this.$forceUpdate()
        })
        return placeholder(h,rawChild)
      }else if(mode === 'in-out'){
        // 新元素先进行过渡,完成后当前元素过渡离开
        if(isAsyncPlaceholder(child)){
          return oldRawChild
        }
        let delayedLeave
        const performLeave = () => {delayedLeave()}
        // 为当前dom上的状态添加hook函数
        mergeVNodeHook(data,'afterEnter',performLeave)
        mergeVNodeHook(data,'enterCancelled',performLeave)
        mergeVNodeHook(data,'delayLeave',leave => {delayedLeave = leave})
      }
    }
    return rawChild
  }
}
```

#### transition-group 

```js
export default {
  props,
  beforMount(){
    const update = this._update
    // transtionGroup挂载之前,重载_update,在dom更新之前,比对group下元素进行更新
    this._update = (vnode,hydrating) => {
      const restoreActiveInstance = setActiveInstance(this)
      this.__patch__(this._vnode,this.kept,false,true)
      this._vnode = this.kept
      restoreActiveInstance()
      update.call(this,vnode,hydrating)
    }
  },
  render(h){
    const tag = this.tag || this.$vnode.data.tag || 'span'
    const map = Object.create(null)
    // 获取group下的vnode
    const prevChildren = this.prevChildren = this.children
    // 获取group下的dom
    const rawChildren = this.$slots.default || []
    const children = this.children = []
    // 获取group的transition过渡状态
    const transitionData = extractTransitionData(this)
    for(let i = 0; i < rawChildren.length; i ++){
      const c = rawChildren[i]
      if(c.tag){
        if(c.key !== null && String(c.key).indexOf('_vlist') !== 0){
          // 为当前group下的每个元素添加transition过渡状态
          children.push(c)
          map[c.key] = c
          (c.data || (c.data = {})).transition = transitionData
        }else{
          ...// 针对transitionGroup中多个元素没有key值的处理
        }
      }
    }
    if(prevChildren){
      const kept = []
      const removed = []
      // 比对vnode和实际dom,需要保留和删除的vnode
      for(let i = 0; i < prevChildren.length; i ++){
        const c = prevChildren[i]
        c.data.transition = transitionData
        c.data.pos = c.elm.getBoundingClientRect()
        if(map[c.key]){
          kept.push(c)
        }else{
          removed.push(c)
        }
      }
      this.kept = h(tag,null,kept)
      this.removed = removed
    }
    return h(tag,null,children)
  },
  updated(){
    const children = this.prevChildren
    const moveClass = this.moveClass || ((this.name || 'v') + '-move')
    if(!children.length || !this.hasMove(children[0].elm,moveClass)){
      return 
    }
    // 根据children的状态进行位置变化
    children.forEach(callPendingCbs)
    children.forEach(recordPosition)
    children.forEach(applyTranslation)
    this._reflow = document.body.offsetHeight
    // 判断group下元素的变化,添加transition事件在事件结束后,移除transition变化class,同时移除事件
    children.forEach((c) => {
      if(c.data.moved){
        const el = c.elm
        const s = el.style
        addTransitionClass(el,moveClass)
        s.transform = s.WebkitTransform = s.transitionDuration = ''
        el.addEventListener(transitionEndEvent,el._moveCb = function cb(e){
          if( e && e.target !== el){
            return 
          }
          if(!e || /transform$/.test(e.propertyName)){
            el.removeEventListener(transitionEndEvent,cb)
            el._moveCb = null
            removeTransitionClass(el,moveClass)
          }
        })
      }
    })
  }
}
```


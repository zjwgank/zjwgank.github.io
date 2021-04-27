---
title: Vue源码解析四
date: 2021-04-27 22:19:18
tags:
---
```js
// 替换Web环境下的Directives,Components
extend(Vue.options.directives,platformDirectives)
```

### platformDirectives

#### bind

只调用一次,指令第一次绑定到元素时调用.在这里可以进行一次性的初始化设置

```js
  bind(el,{value},vnode){
    // 定位el对应的vnode
    vnode = locateNode(vnode)
    // 获取vnode上绑定的transition变化
    const transition = vnode.data && vnode.data.transition
    // 获取el上的display样式
    const originalDisplay = el._vOriginalDisplay = el.style.display === 'none' ? '' : el.style.display
    // 当el上挂载着数据同时存在着数据变换时的transition变化
    if(value && transition){
      // 绑定元素,触发transiton的进入时变化
      vnode.data.show = true
      enter(vnode,() => {
        el.style.display = originalDisplay
      })
    }else{
      // 不存在transition变换时,根据是否有数据变换dom的展示
      el.style.display = value ? originalDisplay : 'none'
    }

  }
```

#### inserted

被绑定的元素插入父节点时调用(仅保证父节点存在,但不一定保证已被插入到文档中)

```js
inserted(el,binding,vnode,oldVnode){
  if(vnode.tag === 'select'){
      // 插入的节点是个select
      if(oldVnode.elm && !oldVnode.elm._vOptions){
        // 如果旧节点Options不存在,则进行更新
        mergeVNodeHook(vnode,'postpatch',() => {
          componentUpdated(el,binding,vnode)
        })
      }else{
        // 设置select选中
        setSelected(el,binding,vnode.context)
      }
  }else if(vnode.tag === 'textarea' || isTextInputType(el.type)){
    // 插入的节点时textarea或者input的一些类型
    // 在el上挂载指令的修饰符
    el._vModifiers = binding.modifiers
    if(!binding.modifiers.lazy){
      // 为inputDom绑定输入法的开始结束事件和change事件
      el.addEventListener('compositionstart',onCompositionStart)
      el.addEventListener('compositionend',onCompositionEnd) // 输入法结束,触发input事件
      el.addEventListener('change',onCompositionEnd)
      if(isIE9){
        el.vmodel = true
      }
    }

  }
}
```

#### update

所在组件的VNode更新时调用,但是可能发生在其子VNode更新之前

```js
update(el,{value,oldValue},vnode){
  // 更新前后,值的状态一样(存在或不存在),el不做变动
  if(!value === !oldValue) return
  vnode = locate(vnode)
  const transition = vnode.data && vnode.data.transition
  // 值的状态发生改变时
  if(transition){
    vnode.data.show = true
    // 如果更新后,值存在,触发transiton-enter状态
    if(value){
      enter(vnode,() => {
        el.style.display = el._vOriginalDisplay
      })
    }else{
      // 如果更新后,值不存在,触发transiton-leave状态
      leave(vnode,() => {
        el.style.display = 'none'
      })
    }
  }else{
    // 不存在transition,根据value是否存在展示
    el.style.display = value ? el._vOriginalDisplay : 'none'
  }
}
```

#### componentUpdated

指令所在组件的VNode及其子Node全部更新后调用

```js
componentUpdated(el,binding,vnode){
  if(vnode.tag === 'select'){
    // 设置select选项
    setSelected(el,binding,vnode.context)
    const prevOptions = el._vOptions
    const curOptions = el._vOptions = [].map.call(el.options,getValue)
    // 更新前后,options是否发生变更,选项是否发生变更
    if(curOptions.some((o,i)=> !looseEqual(o,prevOptions[i]))){
      const needReset = el.multiple ? binding.value.some(v => hasNoMatchingOption(v,curOptions)) : binding.value !== binding.oldValue && hasNoMatchingOption(binging.value,curOptions)
      // 发生改变,触发select的change事件
      if(needReset){
        trigger(el,'change')
      }
    }
  }
}
```

#### unbind

只调用一次,指令与元素解绑时调用

```js
unbind(el,binding,vnode,oldVnode,isDestroy){
  // 指令解绑,判断是否destroy,恢复el的显示状态
  if(!isDestroy){
    el.style.display = el._vOriginalDisplay
  }
}
```
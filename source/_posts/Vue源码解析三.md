---
title: Vue源码解析三
date: 2021-04-23 23:02:17
tags:
---
```js
Vue.prototype.__patch__ = inBrowser ? patch : noop
```
判断当前的环境是否是浏览器下,如果在浏览器下则`patch`,否则使用一个`noop`空函数

```js
// 获取web平台下的document操作Element的方法
import * as nodeOps from 'web/runtime/node-ops'
import { createPatchFucntion } from 'core/vdom/patch'
//  获取公共的ref和directive的比对方法和钩子函数
import baseModules from 'core/vdom/modules/index'
// 获取web环境下的attr、class、style、dom-prop、events、transition的比对方法
import platforModules from 'web/runtime/modules/index'

const modules = platforModules.concat(baseModules)

export const patch = createPatchFunction({nodeOps,modules})
```

#### createPatchFunction

```js
const hooks = ['create','activate','update','remove','destroy']
export function createPatchFunction(backend){
  let i,j;
  const cbs = {}
  const {modules,nodeOps} = backend
  // 针对不同的生命周期,将各个模块的不同生命周期的方法存储在cbs中
  for(i = 0;i < hoos.length; ++ i){
    cbs[hooks[i]] = []
    for(j = 0; j < modules.length; j ++){
      if(isDef(modules[j][hooks[i]])){
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }
  ...
  return function patch(oldVnode,vnode,hydrating,removeOnly){
    // 如果当前vnode不存在,则判断删除原节点
    if(isUndef(vnode)){
      if(isDef(oldVnode)) invokeDestroyHook(oldVnode)
    }
    let isInitialPatch = false
    const insertedVnodeQueue = []
    // 如果原节点不存在,那么根据vnode创建节点插入
    if(isUndef(oldVnode)){
      isInitialPatch = true
      createElm(vnode,insertedVnodeQueue)
    }else{
      const isRealElement = isDef(oldVnode.nodeType)
      if(!isReadElement && sameVnode(oldVnode,vnode)){
        // 比对新旧节点,重新渲染
        patchVnode(oldVnode,vnode,insertedVnodeQueue,null,null,removeOnly)
      }else{
        if(isRealElement){
          ...// 如果是一个真实的dom,判断是否是SSR渲染,并进行处理
        }
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldVnode)
        createElm(vnode,insertedVnodeQueue,oldElm._leaveCb ? null : parentElm,nodeOps.nextSlibing(oldElm))
        // 为vnode的父节点更新信息
        if(isDef(vnode.parent)){
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while(ancestor){
            for(let i = 0; i <cbs.destory.length; ++ i){
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if(patchable){
              for(let i = 0; i < cbs.create.length; i ++){
                cbs.create[i](emptyNode,ancestor)
              }
              const insert = ancestor.data.hook.insert
              if(insert.merged){
                for(let i = 0;i < insert.fns.length; i ++){
                  insert.fns[i]()
                }
              }
            }else{
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // 删除旧节点
        if(isDef(parentElm)){
          removeVnodes([oldVnode],0,0)
        }else if(isDef(oldVnode.tag)){
          invokeDestroyHook(oldVnode)
        }
      }
    }
    // 执行节点插入
    invokeInsertHook(vnode,insertedVnodeQueue,isInitialPatch)
    return vnode.elm
  }
}
```

#### invokeDestoryHook

```js
function invokeDestroyHook(vnode){
  let i,j;
  const data = vnode.data
  if(isDef(data)){
    // 判断是否存在destroy钩子,进行节点删除
    if(isDef(i = data.hook) && isDef(i = i.destroy)) i(vnode)
    // 依此删除vnode上directive,ref,attrs,class等
    for(i = 0;i< cbs.destory.length;i++) cbs.destory[i](vnode)
  }
  // 判断vnode存在子节点,进行子节点销毁
  if(isDef(i = vnode.children)){
    for(j = 0; j < vnode.children.length; j ++){
      invokeDestroyHook(vnode.children[j])
    }
  }
}
```

#### createElm

```js
function createElm(vnode,insertedVnodeQueue,parentElm,refElm,nested,ownerArray,index){
  if(isDef(vnode.elm) && isDef(ownerArray)){
    //  克隆vnode,用作备份或者缓存
    vnode = ownerArray[index] = cloneVNode(vnode)
  }
  vnode.isRootInsert = !nested
  // 如果有子组件,进行子组件构建
  if(createComponent(vnode,insertedVnodeQueue,parentElm,refElm)){
    return
  }
  const data = vnode.data
  const children = vnode.children
  const tag = vnode.tag
  if(isDef(tag)){
    ... // 针对vnode的标签判断与处理
  // 创建节点,设置样式
  vnode.elm = vnode.ns ? nodeOps.createElementNS(vnode.ns,tag) : nodeOps.createElement(tag,vnode)
  setScope(vnode)
  if(__WEEX__){
    ...// 针对weex的处理
  }else{
    // 创建子节点,挂载相关插件,插入到父节点下
    createChildren(vnode,children,insertedVnodeQueue)
    if(isDef(data)){
      invokeCreateHooks(vnode,insertedVnodeQueue)
    }
    insert(parentElm,vnode.elm,refElm)
  }
  }else if(isTrue(vnode.isComment)){ // 创建节点,插入
    vnode.elm = nodeOps.createComment(vnode.text)
    insert(parentElm,vnode.elm,refElm)
  }else{ // 创建节点,插入
    vnode.elm = nodeOps.createTextNode(vnode.text)
    insert(parentElm,vnode.elm,refElm)
  }
}
```

#### patchVnode

```js
function patchVnode(oldVnode,vnode,insertedVnodeQueue,ownerArray,index,removeOnly){
  if(oldVnode === vnode){
    return // 如果节点对比一致,不做操作进行返回
  }
  if(isDef(vnode.elm) && isDef(ownerArray)){
    vnode = ownerArray[index] = cloneVnode(vnode) // 克隆vnode做缓存或备用
  }
  const elm = vnode.elm = oldVnode.elm
  if(isTrue(oldVnode.isAsyncPlaceholder)){
    // 重新渲染
    if(isDef(vnode.asyncFactory.resolved)){
      hydrate(oldVnode.elm,vnode,insertedVnodeQueue)
    }else{
      vnode.isAsyncPlaceholder = true
    }
    return
  }
  // 针对静态dom树复用
  if(isTrue(vnode.isStatic) && isTrue(oldVnode.isStatic) && vnode.key === oldVnode.key && (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))){
    vnode.componentInstance = oldVnode.componentInstance
    return
  }
  let i
  const data = vnode.data
  // 比对新旧节点上的挂件
  if(isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)){
    i(oldVnode,vnode)
  }
  const oldCh = oldVnode.children
  const ch = vnode.children
  if(isDef(data) && isPatchable(vnode)){
    // 更新新旧节点上的挂件
    for(i = 0;i < cbs.update.length; ++ i)cbs.update[i](oldVnode,vnode) 
    // 更新节点
    if(isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode,vnode)
  }
  if(isUndef(vnode.text)){
    if(isDef(oldCh) && isDef(ch)){
      // 比对子节点,进行更新
      if(oldCh !== ch) updateChildren(elm,oldCh,ch,insertedVnodeQueue,removeOnly)
    }else if(isDef(ch)){
      if(process.env.NODE_ENV !== 'production'){
        checkDuplicateKeys(ch)
      }
      // 新节点需要清除text
      if(isDef(oldVnode.text)) nodeOps.setTextContext(elm,'')
      // 添加新节点
      addNodes(elm,null,ch,0,ch.length - 1,insertedVnodeQueue)
    }else if(isDef(oldCh)){
      // 移除旧节点
      removeVnodes(oldCh,0,oldCh.length - 1)
    }else if(isDef(oldVnode.text)){
      // 旧节点text清除
      nodeOps.setTextContentText(elm,'')
    }
  }else if(oldVnode.text !== vnode.text){
    // 更新text
    nodeOps.setTextContent(elm,vnode.text)
  }
  if(isDef(data)){
    if(isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode,vnode)
  }
}
```

#### hydrate

```js
function hydrate(elm,vnode,insertedVnodeQueue,inVPre){
  let i
  const { tag, data, children } = vnode
  inVPre = inVPre|| (data && data.pre)
  vnode.elm = elm
  ...
  // 构建子组件
  if(isDef(data)){
    if(isDef(i = data.hook) && isDef(i = i.init)) i(vnode,true)
    if(isDef(i = vnode.componentInstance)){
      initComponent(vnode,insertedVnodeQueue)
      return true
    }
  }
  if(isDef(tag)){
    if(isDef(children)){
      // 判断元素上没有子节点,构建子节点
      if(!elm.hasChildreNodes()){
        createChildren(vnode,children,insertedVnodeQueue)
      }else{
        if(isDef(i = data) && isDef(i = i.domProps) && isDef(i = i.innerHTML)){
          ...
        }else{
          let childrenMatch = true
          let childNode = elm.firstChild
          // 遍历子节点是否可以匹配上
          for(let i = 0; i < children.length; i ++){
            if(!childNode || !hydrate(childNode,children[i],insertedVnodeQueue,inVPre)){
              childrenMatch = false
              break
            }
            childNode = childNode.nextSibling
          }
          // 实际dom与当前节点不匹配
          if(!childenMatch || childNode){
            ...
          }
        }
      }
    }
    if(isDef(data)){
      let fullInvoke = false
      // 判断全部插件是否挂载上
      for(const ket in data){
        if(!isRenderedModule(key)){
          fullInvoke = true
          invokeCreateHooks(vnode,insertedVnodeQueue)
          break
        }
      }
      ...
    }
  }else if(elm.data !== vnode.text){
    // tag不存在,则直接替换数据
    elm.data = vnode.text
  }
  return true
}
```
---
title: Vue源码解析十三
date: 2021-05-18 22:33:37
tags:
---

### baseModule

#### ref

```js
export default {
  // vnode创建,注册ref
  create(_,vnode){
    registerRef(vnode)
  },
  // vnode更新,ref更新
  update(oldVnode,vnode){
    if(oldVnode.data.ref !== vnode.data.ref){
      registerRef(oldVnode,true)
      registerRef(vnode)
    }
  },
  // vnode销毁,销毁ref
  destroy(vnode){
    registerRef(vnode,true)
  }
}

function registerRef(vnode,isRemoval){
  // 获取vnode上ref
  const key = vnode.data.ref
  if(!isDef(key)) return
  // 获取vm实例
  const vm = vnode.context
  // 获取挂载的节点
  const ref = vnode.componentInstance || vnode.elm
  // 当前vm实例上的所有ref
  const refs = vm.$refs
  // 删除ref
  if(isRemoval){
    if(Array.isArray(refs[key])){
      remove(refs[key],ref)
    }else if(refs[key] === ref){
      refs[key] = undefined
    }
  }else{
    // 判断是否循环进行添加
    if(vnode.data.refInFor){
      if(!Array.isArray(refs[key])){
        refs[key] = ref
      }else if(refs[key].indexOf(ref) < 0){
        refs[key].push(ref)
      }
    }else{
      // 添加ref到refs
      refs[key] = ref
    }
  }
}
```

#### directives

```js
export default{
  create:updateDirectives,
  update:updateDirectives,
  destroy:function unbindDirectives(vnode){
    updateDirectives(vnode,emptyNode)
  }
}
function updateDirectives(oldVnode,vnode){
  // 根据节点上的指令进行节点更新
  if(oldVnode.data.directives || vnode.data.directives){
    _update(oldVnode,vnode)
  }
}

function _update(oldVnode,vnode){
  // 判断节点是否被创建或删除
  const isCreate = oldVnode === emptyNode
  const isDestroy = vnode === emptyNode
  // 获取节点的指令
  const oldDirs = normalizeDirectives(oldVnode.data.directives,oldVnode.context)
  const newDirs = normalizeDirectives(vnode.data.directives,vnode.context)
  const dirsWithInsert = []
  const dirsWithPostpatch = []
  let key,oldDir,dir
  for(key in newDirs){
    oldDir = oldDirs[key]
    dir = newDirs[key]
    // 新指令
    if(!oldDir){
      // 绑定新指令
      callHook(dir,'bind',vnode,oldVnode)
      //  插入新指令
      if(dir.def && dir.def.inserted){
        dirsWithInsert.push(dir)
      }
    }else{
      // 更新指令
      dir.oldValue = oldDir.value
      dir.oldArg = oldDir.arg
      callHook(dir,'update',vnode,oldVnode)
      if(dir.def && dir.def.componentUpdated){
        dirsWithPostpatch.push(dir)
      }
    }
  }
  if(dirsWithInsert.length){
    const callInsert = () => {
      for(let i = 0;i < dirsWithInsert.length; i++){
        callHook(dirsWithInsert[i],'inserted',vnode,oldVnode)
      }
    }
    // 判断如果是新增vnode则执行插入+插入后钩子函数
    if(isCreate){
      mergeVnodeHook(vnode,'insert',callInsert)
    }else{
      callInsert()
    }
  }
  // 执行componentUpdated钩子
  if(dirsWithPostpatch.length){
    mergeVnodeHook(vnode,'postpatch',() => {
      for(let i = 0; i < dirsWithPostpatch.length; i ++){
        callHook(dirsWithPostpatch[i],'componentUpdated',vnode,oldVnode)
      }
    })
  }
  if(!isCreate){
    // 判断指令移除执行移除操作
    for(key in oldDirs){
      if(!newDirs[key]){
        callHook(oldDirs[key],'unbind',oldVnode,oldVnode,isDestroy)
      }
    }
  }
}
```

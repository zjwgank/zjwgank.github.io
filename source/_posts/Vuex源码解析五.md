---
title: Vuex源码解析五
date: 2021-05-30 21:59:31
tags:
---

### createLogger 

```js
// dev环境触发createLogger
export default function createLogger({collapsed = true,filter = (mutation,stateBefore,stateAfter) =>true,transformer = state => state,mutationTransformer = mut => mut,actionFilter = (action,state) =>true,actionTransformer = act => act,logMutations = true,logActions = true,logger = console} = {}){
  return store => {
    // 获取此时state
    let prevState = deepCopy(store.state)
    // 校验logger是否存在
    if(typeof logger === 'undefined'){
      return 
    }
    if(logMutations){
      // 监听所有的mutations
      store.subscribe((mutation,state) => {
        // 获取此时的state
        const nextState = deepCopy(state)
        // dev环境默认true或者不传参数为true
        if(filter(mutation,prevState,nextState)){
          // 获取当前格式化时间 
          const formattedTime = getFormattedTime()
          // 格式化mutation
          const formattedMutation = mutationTransformer(mutation)
          const message = `mutation ${mutation.type}${formattedTime}`
          // 开始记录log
          startMessage(logger,message,collapsed)
          // 记录prevState
          logger.log('%c prev state','color:#9E9E9E;font-weight:bold',transformer(prevState))
          // 记录 当前操作的mutation
          logger.log('%c mutation','color:#03A9F4;font-weight:bold',formattedMutation)
          // 记录当前state
          logger.log('%c next state','color:#4CAF50;font-weight:bold',transformer(nextState))
          // 结束mutation log记录
          endMessage(logger)
        }
        // 监听结束,替换状态
        prevState = nextState
      })
    }
    if(logActions){
      store.subscribeAction((action,state) =>{
        if(actionFilter(action,state)){
          // 因action不直接操作state,此处不记录
          // 记录action的操作时间
          const formattedTime = getFormattedTime()
          // 格式化action
          const formattedAction = actionTransformer(action)
          const message = `action ${action.type}${formattedTime}`
          // 开始action log
          startMessage(logger,message,collapse)
          logger.log('%c action','color:#03A9F4;font-weight:bold',formattedAction)
          // 结束action log
          endMessage(logger)
        }
      })
    }
  }

}
```
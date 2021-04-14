---
title: Vue源码解析一
date: 2021-04-13 22:56:49
tags:
---
### Vue源码build

```js
// package.json
{
  ...
  scripts:{
    "build":"node script/build.js"
  }
  ...
}
// build.js
// 创建dist目录
if (!fs.existSync('dist')){
  fs.mkdirSync('dist')
}
// 获取配置
let builds = require('./config').getAllBuilds()

build(builds) // 依此执行build,输出vue的build文件
function build(builds){
  ...
  const next = () => {
    buildEntry(builds[built]).then(() => {
      built ++
      if(built < total){
        next()
      }
    }).catch(logError)
  }
}
// 根据build的配置,执行rollup,并根据环境判断是否执行压缩
function buildEntry(config){
  ...
  if(isProd){
    const minified = (banner ? banner + '\n' : '') + terser.minify(code,{
      toplevel:true,
      output:{
        ascii_only:true
      },
      compress:{
        pure_funcs:['makeMap']
      }
    }).code
    return write(file,minified,true)
  }else{
    return write(file,code)
  }
}
// 向目标位置写入文件判断是否执行压缩.
function write(dest,code,zip){
  ...
}

// config.js
...
const builds = {
  ... // 各种环境源码的build配置
}
function getConfig(name){
  ... // 为各种build配置格式化,并添加插件
}
// scripts下其他文件提供引用或者辅助函数
// build.js
const builds = {
  'web-runtime-cjs-dev':{
    entry:resolve('web/entry-runtime.js')
    ...
  }
}
// alias.js 提供alias
moudle.exports = {
  web:resolve('src/platforms/web')
  ...
}
// feature-flags.js // 添加plugins配置
// gen-release-note.js 生成release.md
// get-weex-version.js 获取wx环境下build的版本
// verify-commit-msg.js 校验commit信息
```

### Vue源码构成

根据`builds`和`alias`定位至`src/platforms/web/entry-runtime.js`

```js
import Vue from './runtime/index' // 获取Vue的出处
export default Vue // 输出我们平常用的Vue
```

接下来到了`./runtime/index`

```js
import Vue from 'core/index' // Vue类的出处
import Config from 'core/config'
import { extend, noop } from 'shared/util'
import { mountComponent } from 'core/instance/lifecycle'
import { devtools, inBrowser } from 'core/util/index'

import { query, mustUseProp, isReservedTag, isReservedAttr, getTagNamespace, isUnkownElement } from 'web/util/index'
import { patch } from './patch'
import platformDirectives from './directives/index'
import platformComponents from './components/index'
// 为Vue配置web环境下的方法
Vue.config.mustUseProp = mustUseProp
Vue.config.isReservedTag = isReserverTag
Vue.config.isReservedAttr = isReservedAttr
Vue.config.getTagNamespace = getTagNamespace
Vue.config.isUnknownElement = isUnknownElement
// 为Vue配置web环境下的指令和组件
extend(Vue.options.directives,platformDirectives)
extend(Vue.options.components,platformComponents)
// 配置patch
Vue.prototype.__patch__ = inBrowser ? patch : noop
// 配置$mount
Vue.prototype.$mount = function(el,hydrating){
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this,el,hydrating)
}

// dev环境下初始化Vue
if(inBrowser){
  setTimeout(() => {
    if(config.devtools){
      if(devtools){
        devtools.emit('init',Vue)
      }else if(process.env.NODE_ENV !== 'production' && process.env.NODE_ENV !== 'test'){
        console[console.info ? 'info' : 'log']('Download the Vue Devtools extension for a better development experience:\n' +'https://github.com/vuejs/vue-devtools')
      }
    }
    if(process.env.NODE_ENV !== 'production' && process.env.NODE_ENV !== 'test' && config.productionTip !== false && typeof console !== undefined){
      console[console.info ? 'info' : 'log'](
        `You are running Vue in development mode.\n` +
        `Make sure to turn on production mode when deploying for production.\n` +
        `See more tips at https://vuejs.org/guide/deployment.html`
      )      
    }
  },0)
}

export default Vue
```


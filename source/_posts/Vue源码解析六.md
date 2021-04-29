---
title: Vue源码解析六
date: 2021-04-29 22:20:48
tags:
---
```js
// 安装Web环境下的专属方法
Vue.config.mustUseProp = mustUseProp
Vue.config.isReservedTag = isReservedTag
Vue.config.isReservedAttr = isReservedAttr
Vue.config.getTagNamespace = getTagNameSpace
Vue.config.isUnknownElement = isUnknownElement
```

#### mustUseProp

```js
// 标记dom元素必须与哪些属性对应
export const museUseProp = (tag,type,attr)=>{
  return ((attr === 'value' && acceptValue(tag)) && type !== 'button' || (attr === 'selected' && tag === 'option' ) || (attr === 'checked' && tag === 'input') || (attr === 'muted' && tag === 'video'))
}
```

#### isReserverTag

```js
// 标记Vue的预留标签(html标签+svg标签)
export const isReservedTag = (tag) => {
  return isHTMLTag(tag) || isSVG(tag)
}
```

#### isReservedAttr

```js
// 标记Vue标签的预留属性
export const isReservedAttr = makeMap('style,class')
```

#### getTagNameSpace

```js
// 标记标签所属 
export function getTagNameSpace(tag){
  if(isSvg(tag)){
    return 'svg'
  }
  if(tag === 'math'){
    return 'math'
  }
}
```

#### isUnknownElement

```js
// 标记哪些tag为未知元素
const unKnownElementCache = Object.create(null)
export function isUnknownElement(tag){
  if(!inBrowser){
    return true
  }
  if(isReservedTag(tag)){
    return false
  }
  tag = tag.toLowerCase()
  if(unKnownElementCache[tag] !== null){
    return unKnownElementCache[tag]
  }
  const el = document.createElement(tag)
  if(tag.indexOf("-") > -1){
    return (unKnownElementCache[tag] = (
      el.constructor === window.HTMLUnknownElement || el.constructor === window.HTMLElement
    ))
  }else{
    return (unKnownElementCache[tag] = /HTMLUnknownElement/.test(el.toString()))
  }
}
```
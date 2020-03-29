---
title: react源码解析学习
date: 2019-05-04 10:03:56
tags: react
---

react源码深度解析的学习笔记。

<!-- more -->
## 初识
源码地址: [https://github.com/facebook/react](https://github.com/facebook/react)

目录结构：
![category](react/react-category1.png)

React 和 React-Dom 关系
React 和 React-Dom压缩起来也也有一百k左右，大部分源码主要在React-Dom里。React本身是一个定义，定义节点合表现行为的包。具体怎么在DOM上去渲染、更新操作是与平台相关的，在React-Dom和React-Native里是不一样的。所以这部分代码都放在和平台相关的逻辑里。

Flow Type
静态检查工具。文档： [https://flow.org/](https://flow.org/)


jsx -> js转换过程
![jsx](react/jsx.png)

自定义组件必须大写开头
小写的会被认为是原生dom节点，然后又因为是不存在的dom节点，就会报错了。

ReactElement
`export function createElement(type, config, children) {...}`
> type: 节点类型
config: attr,属性
children： 标签中间放的内容

未完。。。

<!--
Component
`import {Component, PureComponent} from './ReactBaseClasses';`

createRef & ref
获取某个dom节点的实例，就可以利用ref
1.string ref
2.function
3.createRef

forwardRef
1. -->
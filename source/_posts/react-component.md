---
title: react-component
date: 2018-06-19 23:41:59
tags:
- react
- typescript
- webpack
---

## react组件的一般般实践，用了typescript、webpack4，react-router4

### 事情是这样的
三周前(5.29)，我决定用react，webpack4，typescript写一个Transfer组件(类antd的穿梭框)，一拖再拖，就前天才写完个粗略版。
今天做个小总结，记录下遇到的问题和改进。

<!-- more -->

先摆个效果图：
<img src="https://github.com/sameenzm/React-Practice/blob/master/screenshots/transfer.png?raw=true" width="585" height="319"/>

### 初版行动
首先，我一开始的思路是，确定props，必然需要有的就是 dataSource 和 value (后来参考了antd的api，改成同名的dataSource和targetKeys)，再者，我考虑了onChange，想着是不是用户不一定要有onChange方法我也能给实现穿梭框的功能，如果用户需要获取新的targetKeys，我也能返回。所以开始动手：

确定props:
```
const TransferPropTypes = {
  dataSource: PropTypes.array,
  targetKeys: PropTypes.array,
  onChange: PropTypes.func,
  className: PropTypes.string
};
```

并且定义了typescript泛型：
```
export interface TransferProps {
  dataSource?: TransferItem[],
  targetKeys?: string[],
  className?: string,
  onChange?: (targetKeys: string[]) => void
};
```

确定state:
```
state = {
  target: Array(0),   // 时刻变换的target，(在Mount的时候去赋值targetKeys)
  tempAdd: Array(0),  // 左框中用户的暂时勾选
  tempDel: Array(0),  // 右框中用户的暂时勾选
  leftData: Array(0), // 左穿梭框中的数据
  rightData: Array(0) // 右穿梭框中的数据
 }
 ```

接着我定义了一个 filterDataSource 方法，传dataSource、target和isLeft的布尔值，返回相应的数据展现数据(isLeft决定是左框数据还是右框数据)
接着就是处理左框和右框的选中事件
再来就是处理向左添加和向右添加的事件
至此就大概完成数据穿梭效果了。


### 发现问题
后来被发现，犯了一个致命错误，我用了内部state： target，是和外部props的targetKeys是一样的值，那么，我就得关注state和props的同步问题，但是我忽略了，以至于是有bug的。解决state和props的同步问题那就得用到componentWillReceiveProps和shouldComponentUpdate等生命周期方法，其实这么一类，渲染流就更乱了。

所以，我就不该用state存target，就直接用props就好。【后来想了一下，我其实心里是有概念的，知道能不用内部state就不要用，遵循单向数据流原则，但是我最后还是感觉迫不得已就用了，其实是因为我一开始思路就想错了，我不可能开发一个用户不处理onChange的组件，应该就是让用户必须有onChange去改targetKeys，保证单一的数据流渲染流】

[初版比较垃圾的代码┭┮﹏┭┮](https://github.com/sameenzm/React-Practice/blob/3e1b8f91bf688e2a526b301632c0d6247df2197b/components/tansfer-select/index.tsx)


### 大改版一下
之后，重写改进了，
props不变，state改进：
```
state = {
  leftSelected: Array(0),
  rightSelected: Array(0),
}
```
改进命名，只能两个state，state和props不再有正交，不需要关心state和props同步的问题
干掉filterDataSource方法，把计算逻辑放在render里，左框数据和右框数据就由dataSource和targetKeys计算决定，就循环渲染那么两次
其他功能性的就同理了
改进之后代码量少了很多，而且渲染流更清晰了，性能也比原来好很多。
[第二版代码](https://github.com/sameenzm/React-Practice/blob/master/components/tansfer-select/index.tsx)


现在完成的版本还是非常不完善的，我会继续更新！


### 个人总结：
1、我一开始写的组件，用了state去存target，和props的targetKeys重复了，违反了单一渲染流！我得时刻关注state和props的同步问题，就非常容易出错！（就得用到componentWillReceiveProps和shouldComponentUpdate等生命周期方法去搞，流越多，越复杂，越难维护）
2、so，state & props 不可以存在相互推导关系，更不可以一模一样！state必须是正交的（正交就是不相交），，否则，就是设计有问题
3、tempAdd和tempDel这种命名不好，最好别简写，因为javascript的根基里就没有简写文化，比如document.getElementById，这么长也不简写。so，命名也尽量表达清楚，别简写，so我改名为：leftSelected 和 rightSelected
4、不要利用componentDidMount、componentWillMount等来赋初始值，这样会引发重新render。初始值，在constructor的时候赋值就好了
5、当if和else的内容都很多的时候，就拆分成两个function，颗粒度拆小点，因为放在一起，需要改动的时候，涉及的面积越广，就越容易出错。






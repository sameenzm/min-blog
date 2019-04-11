---
title: 初识 redux-saga
date: 2019-04-11 20:52:44
tags:
- redux middleware
- async

---

最近项目用了dva，dva对于异步action的处理是用了redux-saga，故简单学习了下redux-saga；

<!--more -->

#### 目录
1. [是什么](#是什么)
	1. [名词解释](#名词解释)
1. [为什么](#为什么)
	1. [作用](#作用)
	1. [优点](#优点)
1. [怎么用](#怎么用)
	1. [hello saga](#hello saga)
	1. [api](#api)
	1. [高级概念](#高级概念)
	1. [对比redux-thunk](#对比redux-thunk)


## 是什么

redux-saga 就是 redux 的一个**中间件**，用于**更优雅地管理副作用**(side effects);


redux-saga可以理解为一个 和 系统交互的 常驻进程，可简单定义：
`saga = Worker + Warcher`

#### 名词解释

- side effects
> Side effects are the most common way that a program interacts with the outside world    (people, filesystems, other computers on networks)   [from  Wikipedia]

副作用是 程序与外部世界（人、文件系统，网络上的其他计算机） 交互的最常用的方式。
映射到前端， 副作用一般指异步网络请求。

- Effect
> An effect is a plain JavaScript Object containing some instructions to be executed by the saga middleware.

effect 是一个普通的 javascript对象，包含一些指令，这些指令最终会被 redux-saga 中间件 解释并执行。




在 redux-saga 世界里，所有的 Effect 都必须被 yield 才会执行

原则上来说，所有的 yield 后面也只能跟Effect，以保证代码的易测性。
eg：

`yield fetch(UrlMap.fetchData);`

应该用 call Effect ：
`yield call(fetch, UrlMap.fetchData)`

- task
task 是 generator 方法的执行环境，所有saga的generator方法都跑在task里。


## 为什么

##### 作用


用于**更优雅地管理副作用**， 在前端就是异步网络请求；本质就是为了解决异步action的问题；
##### 优点

- 副作用转移到单独的saga.js中，不再掺杂在action.js中，保持 action 的简单纯粹，又使得异步操作集中可以被集中处理。[对比redux-thunk](#对比redux-thunk)

- redux-saga 提供了丰富的 Effects，以及 sagas 的机制（所有的 saga 都可以被中断），在处理复杂的异步问题上更顺手。提供了更加细腻的[控制流](#登录流程案例)。

- 对比thunk，dispatch 的参数依然是一个纯粹的 action [(FSA)](#对比redux-thunk)。

- 每一个 saga  都是 一个 generator function，代码可以采用 同步书写 的方式 去处理 异步逻辑（No Callback Hell），代码变得更易读。

- 同样是受益于 generator function 的 saga 实现，代码异常/请求失败 都可以直接通过 try/catch 语法直接捕获处理。

## 怎么用

### hello saga
1. 单独的文件：sagas.js, 统一管理副作用：
```
export function* helloSaga() {
   console.log('Hello Sagas!');
}
```
2. 将saga和store关联起来, 入口文件 main.js：
```
 import { createStore, applyMiddleware } from 'redux';
 import createSagaMiddleware from 'redux-saga';

 import helloSaga from './sagas'
 import rootReducer from './reducers'

 // 创建 saga middleware
  const sagaMiddleware  = createSagaMiddleware();

 // 创建 store
 const store = createStore(
   rootReducer,
   applyMiddleware(sagaMiddleware)   // 注入 saga middleware
 );

 // 启动 saga
 sagaMiddleware.run(helloSaga);

 // 省略ReactDOM.render部分的代码...
```

这时候就可以看到`Hello Sagas!`了；

代码分析：
> **line 8** :通过redux-saga提供的工厂函数 createSagaMiddleware 创建 sagaMiddleware（当然创建时，你也可以传递一些可选的[配置参数](https://redux-saga.js.org/docs/api/)）。
**line 11-14** : 创建 store 实例, 并注入 saga中间件。意味着：之后每次执行 store.dispatch(action)，数据流都会经过 sagaMiddleware 这一道工序，进行必要的 “加工处理”（比如：发送一个异步请求）。
**line 17** : 启动 saga，调用run方法使得generator可以开始执行，也就是执行 rootSaga。通常是程序的一些初始化操作（比如：初始化数据、注册 action 监听）。




3、接下来加入异步调用的流程

先看下要实现的效果：

![计数器](https://cube.elemecdn.com/7/3f/bf8593c537d4fb0f6a2d2673fc316gif.gif)




省略UI代码；


reducer中已有加一的处理：
```
...
case 'INCREMENT':
  return {
    ...state,
    count: state.count + 1
}
...
```

sagas.js：

```
 import { all, put, takeEvery } from 'redux-saga/effects'
 const delay = (ms) => new Promise(res => setTimeout(res, ms))

 // worker Saga: 执行异步的 increment 任务
 export function* incrementAsync() {
   yield delay(1000) // middleware 拿到一个 yield 后的 Promise，暂停1s后再继续执行
   yield put({ type: 'INCREMENT' })  // 告诉 middleware 发起一个 INCREMENT 的 action。
 }

 // watcher Saga: 在每个INCREMENT_ASYNC上生成一个新的incrementAsync任务
 export function* watchIncrementAsync() {
  yield takeEvery('INCREMENT_ASYNC', incrementAsync)
 }

// 启动saga们
 export default function* rootSaga() {
  yield all([
    watchIncrementAsync(),
    helloSaga()
  ])
 }
```



把saga和store联系起来的代码和上面相似，就是把helloSaga替换成rootSaga即可；

代码分析：
> Sagas 是被实现为 Generator functions 的
**line 2** ： 创建一个`delay`函数，返回一个Promise，它在指定的毫秒数后解析。
**line 5-8** : incrementAsync 这个 Saga 会暂停，直到 delay 返回的 Promise 被 resolve，即 1000ms 之后；
**line 6** : middleware 拿到一个 yield 后的 Promise，middleware 暂停 Saga，直到 Promise 完成。一旦 Promise 被 resolve，middleware 会恢复 Saga 接着执行，直到遇到下一个 yield。
**line 7** : 这里就是第二个yield啦，这里的 `put({type: 'INCREMENT'})` 就是一个Effect，Effect 是纯js对象，其中包含了给 middleware 执行的指令；当 middleware 拿到被Saga yield的Effect的时候，也会暂停Saga，直到Effect 执行完成，然后Saga 会再次被恢复。
**line 11-13** ： 写一个watcher saga，用redux-saga的api `takeEvery` 来监听所有的 INCREMENT_ASYNC action，并在 action 被匹配时执行 incrementAsync 任务。
**line 15-18** : 有了Saga，，现添加一个rootSaga来负责启动所有Saga，用了`all` api，如果有其他Saga都能一起启动。



line 7 返回的是一个Effect，`console('Effect', put({ type: 'INCREMENT' }))`：
![Effect](https://cube.elemecdn.com/5/42/103157821b9f05daab45d395b3fd4png.png)



基于redux的数据流：
状态决定展现，交互就是改状态
![redux的数据流](https://www.ruphi.cn/usr/uploads/2017/06/1616038871.png)

基于redux-saga的一次完整单向数据流：
![完整单向数据流](https://gw.alipayobjects.com/zos/rmsportal/fkpkHBwmTAJtjgyDhMto.png)



### api

> 在第一次[使用dva](#在dva中使用)的时候，用的最多的api就是`put`和`call`，有时还有用`select`


#### ❀Effect 创建器（creators）

**1、`put(action)`**
>创建一个Effect描述信息，指示 middleware 向Store dispatch一个action

相当于在 saga 中调用 store.dispatch(action)。

**2、`select(selector, ...args)`**
>创建一个Effect，指示 middleware 调用提供的选择器获取 Store state 上的数据，即获取状态

**3、`call(fn, ...args)`**
>创建一个Effect描述信息，指示 middleware 以args为参数调用fn；

即执行fn(...args);
如果fn是个Generator，或者返回Promise，那么会阻塞当前 saga 的执行，直到被调用函数 fn 返回结果，才会执行下一步代码。

**4、`take(pattern)`**
>创建一个Effect描述信息，指示 middleware 等待 Store 上指定的 action。 Generator 会暂停(被阻塞了)，直到一个与 pattern 匹配的 action 被发起。
有种事件监听的感觉。
take的返回值是action

如果调用take而没有参数或是'*'，则所有调度的操作都匹配（例如，`take()`将匹配所有操作）

可以监听多个，eg:
` yield take(['LOGOUT', 'LOGIN_ERROR'])`


**5、`fork(fn, ...args)`**
>创建一个Effect描述信息，指示 middleware 以 无阻塞调用 方式执行 fn
fork的返回值是task

类似于 call effect，区别在于它不会阻塞当前 saga，如同后台运行一般，会立即返回一个 task 对象。
`yield fork(fn ...args)` 的结果是一个 Task 对象 —— 具有一些有用方法和属性的对象。

**6、`cancel(task)`**
> 创建一个Effect描述信息，针对 fork 方法返回的 task ，可以进行取消关闭。

**7、`cancelled()`**
> 创建一个Effect描述信息，指示 middleware 返回 该 generator 是否已经被取消。通常你会在 finally 区块中使用这个 Effect 来运行取消时专用的代码。


#### ❀在强大的低阶 API 之上构建的 wrapper effect

**8、`takeEvery(pattern, saga, ...args)`**
> 被 dispatch 的 action 中，在匹配到 pattern 的每一个 action 上派生一个 saga
takeEvery 是一个使用 take 和 fork 构建的高级 API。

实现：
```
const takeEvery = (patternOrChannel, saga, ...args) => fork(function*() {
  while (true) {
    const action = yield take(patternOrChannel)
    yield fork(saga, ...args.concat(action))
  }
})
```

**9、`takeLastest(pattern, saga, ...args)`**
> 被 dispatch 的 action 中，在匹配 pattern 的每一个 action 上派生一个 saga。并自动取消之前所有已经启动但仍在执行中的 saga 任务。
takeLatest 也是一个使用 take 和 fork 构建的高级 API。
实现：
```
const takeLatest = (patternOrChannel, saga, ...args) => fork(function*() {
  let lastTask
  while (true) {
    const action = yield take(patternOrChannel)
    if (lastTask) {
      yield cancel(lastTask) // 如果任务已经结束，cancel 则是空操作
    }
    lastTask = yield fork(saga, ...args.concat(action))
  }
})
```

#### ❀Effect 组合器（combinators）

**10、`race([...effects]) `**
> 创建一个Effect描述信息，指示 middleware  在多个 Effect 之间运行一个 race(与 Promise.race([...]) 的行为类似)。

race可以取到最快完成的那个结果，常用于请求超时

**11、`all([...effects])`**
>创建一个 Effect 描述信息，指示 middleware 并行运行多个 Effect，并等待它们全部完成。这是与标准的[Promise#all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)相对应的 API。

也可用`[...effects]`，yield 一个包含 effects 的数组，eg：

```
import { call } from 'redux-saga/effects'

// 正确写法, effects 将会同步执行
const [users, repos] = yield [
  call(fetch, '/users'),
  call(fetch, '/repos')
];
```
generator 会被阻塞直到所有的 effects 都执行完毕，或者当一个 effect 被拒绝 （就像 Promise.all 的行为）。


欲了解其他api可以访问： [速查直达](https://redux-saga.js.org/docs/api/)


[返回](#api)
##### 在dva中使用
![dva中使用](https://cube.elemecdn.com/6/9f/b42ac72b9bf5b58a27411859e8682png.png)


对于try catch的[额外补充](#额外补充)



##### Call vs Fork
saga 中 call 和 fork 都是用来执行指定函数 fn，区别在于：
- call Effect 会阻塞当前 saga 的执行，直到被调用函数 fn 返回结果才执行下一步代码。
- fork Effect 则不会阻塞当前 saga，会立即返回一个 task 对象。

>fork 的异步非阻塞特性更适合于在后台运行一些不影响主流程的代码





### 高级概念
1、监听未来的action  —— take


先看下要实现的效果：
![日志记录器](https://cube.elemecdn.com/d/30/2f28cecf36826405206c4781f0d59gif.gif)


take的实现：
```
 import { select, take } from 'redux-saga/effects'

 function* watchAndLog() {
  while (true) {
    const action = yield take('*');
    const state = yield select();
    console.log('action', action);
    console.log('state after', state);
  }
 }
```
代码分析：
> 这是一个简单的打印日志功能
**line 5**: 指示 middleware 等待一个特定的 action。这里整个Generator被暂停了，直到匹配到的action被dispatch了，这里是*，所以是任意一个action; yield take('*')的返回值就是匹配到的action
**line 6**: 用select api 拿到所有状态
**line 4-9**: 这里用了`while(true)`,因为 Generator 函数不具备 从运行至完成 的行为（run-to-completion behavior），这个Generator 会每次迭代到第5行时阻塞，以等待 action 发起。



对比`takeEvery`，实现一样的效果：
```
import { select, takeEvery } from 'redux-saga/effects'

function* watchAndLog() {
  yield takeEvery('*', function* logger(action) { // 这里action被被动注入回调了
    const state = yield select()

    console.log('action', action)
    console.log('state after', state)
  })
}
```
> 可以看出，takeEvery 的实现中， 匹配到action就执行回调， action就被动的被 push 到任务处理函数的。
每次 action 被匹配时任务处理函数就会一遍又一遍地被调用。并且它们也无法控制何时停止监听。
而 take 的实现中，Saga 是自己主动 pull action 的，就像是在执行一个普通函数一样: `action = getNextAction()`。主动拿到action就可以控制停止，流程上更灵活；

eg:
监听用户的操作，并在用户初次创建完三条 Todo 信息时显示祝贺信息
```
import { take, put } from 'redux-saga/effects'

function* watchFirstThreeTodosCreation() {
  for (let i = 0; i < 3; i++) {
    const action = yield take('TODO_CREATED')
  }
  yield put({type: 'SHOW_CONGRATULATION'})
}
```
> action被匹配到3次之后，Generator 会被回收并且相应的监听不会再发生

主动拉取 action 可以让我们使用熟悉的同步风格来描述我们的控制流
eg:
监听得来，还有顺序
```
function* loginFlow() {
  while (true) {
    yield take('LOGIN')
    // ... perform the login logic
    yield take('LOGOUT')
    // ... perform the logout logic
  }
}
```

[返回](#为什么)

2、无阻塞调用 —— fork


###### 登录流程案例
就着上面说的登录登出流程，先提前看一段代码（有问题的）：
```
 import { take, call, put } from 'redux-saga/effects'
 import Api from '...'

 function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    return token
  } catch(error) {
     yield put({type: 'LOGIN_ERROR', error})
  }
 }

 function* loginFlow() {
   while(true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    const token = yield call(authorize, user, password)
    if(token) {
      yield call(Api.storeItem({token}))
      yield take('LOGOUT')
      yield call(Api.clearItem('token'))
    }
   }
 }
```
> line 16： 当 LOGIN_REQUEST 的action被匹配时，拿到用户名密码就去调用 authorize 这个Generator
（PS: call 不仅可以用来调用返回 Promise 的函数。我们也可以用它来调用其他 Generator 函数。 ）
line 4-12： 拿到用户名密码之后就去执行真正的请求，这时候 authorize 就被阻塞了，等待着拿token；拿到 token 就 dispatch 登录成功，返回token；登录失败就 dispatch 登录失败
line 18-22： 登录成功之后就缓存token，并且监听登出的action，当匹配LOGOUT，则清楚token

*上面的代码流程很清晰，就像阅读同步代码一样，自然顺序确定了执行步骤，不用专门理解控制流（如果用takeEvery就会需要去理解）*

**但是**，上面的代码有问题。
当用户点登录之后，authorize 被阻塞，请求还没返回，token还没拿到，就在此刻，用户又点了登出，那么...

![有问题的登录流程](https://cube.elemecdn.com/e/5e/e6ba777f34a42d4be7a36ee959da2png.png)


*上面代码的问题是 call 是一个会阻塞的 Effect。即 Generator 在调用结束之前不能执行或处理任何其他事情*，然后，LOGOUT 与调用 authorize 是 并发的，导致出问题了


所以，需要本小节的主角登场 —— **☆ fork ☆**

fork 一个 任务，任务会在后台启动，Generator不会被阻塞，调用者可以继续它自己的流程，而不用等待被 fork 的任务结束。

具体改进如下：
```
 import { fork, call, take, put, cancel } from 'redux-saga/effects'
 import Api from '...'

 function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    yield call(Api.storeItem, {token})
  } catch(error) {
     yield put({type: 'LOGIN_ERROR', error})
  } finally {
     if (yield cancelled()) {
       // 取消task之后的操作，比如取消loading之类
     }
   }
 }

 function* loginFlow() {
   while (true) {
     const {user, password} = yield take('LOGIN_REQUEST')
     const task = yield fork(authorize, user, password)
     const action = yield take(['LOGOUT', 'LOGIN_ERROR'])
     if (action.type === 'LOGOUT') {
       yield cancel(task)
     }
     yield call(Api.clearItem, 'token')
   }
 }
```

> yield fork 的结果是一个[Task Object](https://redux-saga.js.org/docs/api/index.html#task).
line 21： 改用 fork api 调用 authorize ，loginFlow 就不会被阻塞
line 22：监听 2 个并发的 action，
line 22-26: 会有三种情况：
1、在登出之前，token已经拿到了，那么会 dispatch LOGIN_SUCCESS，就结束了，就算在登出流程也是正常的
2、在登出之前，登录失败了，那么会 dispatch LOGIN_ERROR ，然后清除token，结束；进入另外一个 while 迭代等待下一个 LOGIN_REQUEST
3、token还没拿到，用户就登出了，那 loginFlow 会匹配到 LOGOUT ，取消掉 authorize 处理进程，清除token，然后就等待下一个 LOGIN_REQUEST 了
line 8： 使用 fork 之后就拿不到token了，因为不应该等待它，所以将 token 存储操作移到 authorize 任务内部了
line 11-15:如果task被取消之后，你还需要做一些操作，比如Loading本来是true的，你想改成false，那可以利用`canceled`这个api来确定是否取消了

3、在多个 Effects 之间启动 race
eg:
触发一个远程的获取请求，并且限制了 1 秒内响应，否则作超时处理
```
import { race, call, put } from 'redux-saga/effects'
import { delay } from 'redux-saga'
function* fetchPostsWithTimeout() {
  const {posts, timeout} = yield race({
    posts: call(fetchApi, '/posts'),
    timeout: call(delay, 1000)
  })
  if (posts) {
    put({type: 'POSTS_RECEIVED', posts})
  } else {
    put({type: 'TIMEOUT_ERROR'})
  }
}
```

4、通过`yield*`进行排序
```
function* playLevelOne() { ... }
function* playLevelTwo() { ... }
function* playLevelThree() { ... }

function* game() {
  // 利用 yield* 组织saga的顺序
  const score1 = yield* playLevelOne()  // ※
  yield put(showScore(score1))

  const score2 = yield* playLevelTwo()   // ※
  yield put(showScore(score2))

  const score3 = yield* playLevelThree()   // ※
  yield put(showScore(score3))
}
```

<!-- 5、组合saga -->



更多高级概念，可[直达这里](https://redux-saga.js.org/docs/advanced/)学习


[返回](#为什么)
#### 对比redux-thunk
一般情况下，`action` 都是符合 [FSA](https://github.com/redux-utilities/flux-standard-action) 标准的（即：a plain javascript object），如下：
```
{
  type: 'ADD_TODO',
  payload: {
    text: 'Do something.'
  }
}
```
含义：当执行`dispatch(action)`时，通知`reducer`，并且把`action.payload (新状态数据)` 以`action.type`的方式(操作) **同步更新** 到本地store。

但是，涉及请求的时候，payload一般来自于远程服务端；然后redux-thunk就以 middleware 的形式来增强 redux store 的 dispatch 方法，（即支持 `dispatch(function)`）,看下面代码：
```
// action.js
// -----------------
// 符合 FSA 的 action
export const setReplyModalData = (data) => {
    return { type: SET_REPLY_MODAL_DATA, payload:{data} };
};

// 这个 action return 的是一个function
// function 中包含了业务数据请求代码逻辑
export function fetchData(someValue) {
  return (dispatch, getState) => {
    myAjaxLib.post("/someEndpoint", { data: someValue })
      .then(response => dispatch({ type: "REQUEST_SUCCEEDED", payload: response })
      .catch(error => dispatch({ type: "REQUEST_FAILED", error: error });
  };
}

// component.js
// ------------
// View 层 dispatch(fn) 触发异步请求
// 这里省略部分代码
this.props.dispatch(fetchData({ hello: 'saga' }));

```

同样的代码，redux-saga的实现：
它单独起一个新文件saga.js,然后把异步action迁移到里面
```
// saga.js
// ----------
// worker saga
// 它是一个 generator function
// function 中也包含了业务数据请求代码逻辑，但 是同步的写法
function* fetchData(action) {
  const { payload: { someValue } } = action;
  try {
    const result = yield call(myAjaxLib.post, "/someEndpoint", { data: someValue });
    yield put({ type: "REQUEST_SUCCEEDED", payload: response });
  } catch (error) {
    yield put({ type: "REQUEST_FAILED", error: error });
  }
}

// watcher saga
// 监听每一次 dispatch(action)
// 如果 action.type === 'REQUEST'，那么执行 fetchData
export function* watchFetchData() {
  yield takeEvery('REQUEST', fetchData);
}

// component.js
// -------
// View 层 dispatch(action) 触发异步请求
// 这里的 action 依然可以是一个 plain object
this.props.dispatch({
  type: 'REQUEST',
  payload: {
    someValue: { hello: 'saga' }
  }
});

// action.js
// 然后action里就保持了所有都是符合FSA的action了，更干净
export const setReplyModalData = (data) => {
    return { type: SET_REPLY_MODAL_DATA, payload:{data} };
};

```

综上，redux-saga对比redux-thunk的优点：
- 副作用转移到单独的saga.js中，不再掺杂在action.js中，保持 action 的简单纯粹，又使得异步操作集中可以被集中处理。

- dispatch 的参数依然是一个纯粹的 action (FSA)，而不是充满 “黑魔法” thunk function。

- 每一个 saga  都是 一个 generator function，代码采用 同步书写 的方式来处理 异步逻辑（No Callback Hell），代码变得更易读

- 受益于 generator function 的 saga 实现，代码异常/请求失败 都可以直接通过 try/catch 语法直接捕获处理。




[返回](#在dva中使用)
##### 额外补充


请求都要加上try catch，多考虑，避免网页挂掉；
**那什么时候写具体的catch呢？**
>感觉 如果是要**获取数据**的时候最好写清楚catch，因为这种情况下后端的toast一般都是网络请求失败这种mw，免得拿不到数据就啥也做不了了，这时，作为前端，可以给到用户一个友好的toast。如果后端没返回errmsg，页面也没任何提示就特别不友好如果是**创建、编辑、新增等**，就不需要前端去做toast，在接口统一处去toast后端返回的error message，才可以toast具体的原因，比如群组重名了（这个是需要后端去查库的）？还是什么数据不合法？还是其他原因，就是**提交类型的接口**，在前端能做的表单校验完成之后，还是有接口报错，那是后端才能检查出来的，就toast后端的抛出来的问题；




#### 最后总结一下
- redux-saga就是一个redux的中间件，用于更优雅的管理异步
- redux-saga有一堆的api可供使用
- 可以利用同步的方式处理异步逻辑，便于捕获异常，易于测试；


#### 参考链接

[Redux-Saga Tutorial](https://redux-saga.js.org/)

[Redux-Saga Tutorial中文版](https://redux-saga-in-chinese.js.org/)
[redux-saga 漫谈](https://www.yuque.com/lovesueee/blog/redux-saga#0fitsr)





---

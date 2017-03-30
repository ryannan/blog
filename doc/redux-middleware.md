# Redux 中间件

#### redux 中间件是什么

中间件是一个函数，对 store.dispatch 进行了重定义，在发出 Action 和执行 Reducer 这两步之间，添加了其他扩展功能。

#### 为了解决了什么问题

* 解决了 redux 的扩展性，便于扩展功能
* 解决了异步的问题，有个场景，在发送请求前通知 View 即将发送的状态，等待发送请求成功或者失败后，通知 View 对应成功或者失败的状态，如何才能在发送结束后，自动发出第二个 Action 通知 View 更新对应的状态呢？这即是中间件 redux-thunk 或者 redux-promise 解决的异步的问题。

#### applyMiddleware

```
export default function applyMiddleware(...middlewares) {
  //返回闭包函数，共享store
  return (createStore) => (reducer, preloadedState, enhancer) => {
    var store = createStore(reducer, preloadedState, enhancer)
    var dispatch = store.dispatch
    var chain = []

    var middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    }
    //合并middlewareAPI至middlewares
    chain = middlewares.map(middleware => middleware(middlewareAPI))

    //组合middlewares和store.dispatch，生成新的dispatch
    dispatch = compose(...chain)(store.dispatch)

    //返回新的store,dispatch
    return {
      ...store,
      dispatch
    }
  }
}
```

* 默认的 store.dispatch 是一个对象，applyMiddleware 对 store.dispatch 进行了改造，使其它参数由对象变成了函数，函数的两个参数为 dispatch 和 getState 两个方法。
* 在 createStore 时，如果有中间件，则 createStore 中相关方法（dispatch, getState等）会在applyMiddleware 中执行，从而共享 state, dispatch。如果没有中间件，则常规执行。
* applyMiddleware 将所有中间件组成了一个数组，**依次执行**，中间件内部可以拿到 getState 和 dispatch 两个方法。
* appleMidldeware 返回的是新的 store 和 dispatch
* appleMiddlewares 中的 dispatch 是用 compose 方法处理返回的，compose 通过将中间件函数组合在一起，组合成一个高阶函数（函数式编程）。依次执行，向下个执行的中间件传递 action，这样一个 action 会走遍所有中间件，最后生成一个新的 dispatch。

#### 生命周期

每个中间件有三层，如果有 N 个中间件，compose 执行中间件，从最后一个中间件开始，得到第三层返回的方法，这个方法则是倒数第二个中间件第二层的参数。

* 第一层，获取共享 getState 和 dispatch。
* 第二层，传入上一个执行的中间件的 action 函数，使用 compose 生成一个高阶函数，在最后一个执行的中间件中返回，即为新的 dispatch。可以理解为递归执行每个中间件的第三层。
* 第三层，按照中间件开发约定，实现中间件的功能，返回 action。

redux-thunk 中间件的代码

```
// 第一层方法
function (_ref) {
    ...
    // 第二层
    return function (next) {
        // 第三层
        return function (action) {
            if (typeof action === 'function') {
                return action(dispatch, getState, extraArgument);
            }
            return next(action);
        };
    };
}
```

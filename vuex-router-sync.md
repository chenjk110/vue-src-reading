# vuex-router-sync包源码解读

vuex-router-sync是vue的一个官方包，实现router信息与vuex结合，在vuex中注入路由信息，并且实现双向动态绑定，即:

1. 监听store中的路由信息，同步更新router的信息。
2. 在router的afterEach钩子中，同步更新store中的route信息

整个包的主要的关键点有以下几个：

1. 定义sync同步函数，传入store，router，options实现同步，并返回unsync解除绑定
2. 在vuex的store中注册路由状态模块
3. 监听store中的路由状态变更，同步router
4. 在router的afterEeach钩子中实现同步变更store中的状态
5. 一个cloneRoute的辅助函数，实现route对象信息的提取以及state的生成

```js
/* 1.定义同步API */
exports.sync = function (store, router, options) {

  /* 定义route的store模块名称，可通过'options.moduleName'字段配置，默认为'route' */
  const moduleName = (options || {}).moduleName || 'route'

  /* 2.动态注册路由状态模块 */
  store.registerModule(moduleName, {
    namespaced: true,
    state: cloneRoute(router.currentRoute), // 获取当前路由信息state
    mutations: {
      // 定义路由变更回调，并约定名称为'ROUTE_CHANGED'，
      // 在后面对afterEach路由钩子中实现commit提交变更
      'ROUTE_CHANGED'(state, transition) {
        store.state[moduleName] = cloneRoute(transition.to, transition.from) // 创建新的route状态对象并更新
      }
    }
  })

  let isTimeTraveling = false // 时间穿梭，devtool相关
  let currentPath // 当前路径

  // 3.更新store，则同步router信息
  const storeUnwatch = store.watch(
    state => state[moduleName],
    route => {
      const { fullPath } = route
      if (fullPath === currentPath) {
        return
      }
      if (currentPath != null) {
        isTimeTraveling = true
        router.push(route)
      }
      currentPath = fullPath
    },
    { sync: true }
  )

  // 4.注册路由钩子处理，在每次路由变更后实现同步路由信息到store
  // 返回钩子解除回调
  const afterEachUnHook = router.afterEach((to, from) => {

    // 若是时光穿梭调试，则不更新状态
    if (isTimeTraveling) {
      isTimeTraveling = false
      return
    }

    currentPath = to.fullPath // 获取当前路由完整路径
    store.commit(moduleName + '/ROUTE_CHANGED', { to, from }) // 变更当前路由信息state
  })

  // sync函数执行后，返回unsync回调方法
  return function unsync() {
    // 取消相关afterEach路由钩子处理
    if (afterEachUnHook != null) {
      afterEachUnHook()
    }

    // 取消相关store变更观察器注册
    if (storeUnwatch != null) {
      storeUnwatch()
    }

    // 取消模块注册
    store.unregisterModule(moduleName)
  }
}

/** 5.拷贝route对象信息，返回对应的state格式 */
function cloneRoute(to, from) {
  const clone = {
    name: to.name, // 名称
    path: to.path, // 路径
    hash: to.hash, // hash信息
    query: to.query, // query信息
    params: to.params, // params信息
    fullPath: to.fullPath, // 完整路径
    meta: to.meta // meta信息
  }
  if (from) {
    clone.from = cloneRoute(from) // 递归提取from路由信息
  }
  return Object.freeze(clone) // 调用Object.freeze方法返回一个immutable的路由信息对象
}
```


[TOC]

# vue router 3.x

在Vue中使用是不需要 ```Vue.install(vue-router)``` 挂到vue下的

因为在 [/src/router.js](https://gitee.com/mirrors/vue-router/blob/dev/src/router.js#L292) 中会自己挂上去

```javascript
if (inBrowser && window.Vue) {
    // 这里会调用 VueRouter.install
  window.Vue.use(VueRouter)
}
```

## 如何解析路由参数

## [VueRouter.install](https://gitee.com/mirrors/vue-router/blob/dev/src/install.js)

通过 ```Vue.mixin``` 的方式，在生命周期中做事

```javascript
// 🤔️这样mixin进去不是每个组件都会执行？？
// 🤔️有必要每个组件都执行吗？？
Vue.mixin({
    // 分别在 beforeCreate 和 destroyed 两个声明周期中做点事
    beforeCreate () {
        ...
    },
    destroyed () {
        ...
    }
})
```

### 在 ```beforeCreate``` 中做了什么？


### 在 ```destroyed``` 中做了什么？


## VueRouter 的 [init](https://gitee.com/mirrors/vue-router/blob/dev/src/router.js#L88) 

```javascript
export default class VueRouter {
    ...

    init (app) {
        ...

        // 在这里监听路由变化
        const history = this.history
        if (history instanceof HTML5History || history instanceof HashHistory) {
            ...

            // transitionTo 是从父类 History 继承的
            // 
            history.transitionTo(history.getCurrentLocation(), 
            setupListeners, setupListeners,)
        }

        ...
    }

    ...
}

```

在 构造函数 中调用了 [```createMatcher```](https://gitee.com/mirrors/vue-router/blob/dev/src/create-matcher.js#L19) 函数，通过传入的路由信息，在内部生成对应的路由映射关系，并返回这些方法：  ```match```, ```addRoute```, ```getRoutes```, ```addRoutes``` 

```javascript
export default class VueRouter {
    ...
    matcher
    ...
    constructor (options) {
        ...
        // 在这里调用
        // 传入配置的 routes 信息
        this.matcher = createMatcher(options.routes || [], this)

        ...
    }


    ...
}

```

### [```createMatcher```](https://gitee.com/mirrors/vue-router/blob/dev/src/create-matcher.js#L19)

```javascript
export function createMatcher (routes, router) {
    // 这里调用 createRouteMap 创建路由记录
    // createRouteMap 中会 调用 addRouteRecord 函数向 patchList, pathMap 中添加路由信息
    const { patchList, pathMap, nameMap } = createRouteMap(routes)

    ...
}

```

### [```createRouteMap```](https://gitee.com/mirrors/vue-router/blob/dev/src/create-route-map.js#L7)

调用 ```addRouteRecord``` 函数向 ```pathList, pathMap``` 中添加记录
```addRouteRecord``` 会递归处理 ```routes``` 中的 ```children``` 字段

```javascript
export function createRouteMap (routes, oldPathList, oldPathMap, oldNameMap, parentRoute) {
    ...

    routes.forEach(route => {
        addRouteRecord(pathList, pathMap, nameMap, route, parentRoute)
    })

    ...
}

```


## 如何监听路由变化？

### 监听 hash 变化

在 [```HashHistory```](https://gitee.com/mirrors/vue-router/blob/dev/src/history/hash.js#L10) 里的 [```setupListeners```](https://gitee.com/mirrors/vue-router/blob/dev/src/history/hash.js#L22) 中监听

```javascript
export class HashHistory extends History {
    ...

    setupListeners () {
        ...

        // 当 history 变化时就会触发 popstate 事件
        const eventType = supportsPushState ? 'popstate' : 'hashchange'
        window.addEventListener(eventType, handleRoutingEvent)

        ...
    }

    ...
}
```

但触发这个 ```setupListeners``` 函数的路径有点长😮‍💨

要追溯到初始化的时候

从 VueRouter 的 [init](https://gitee.com/mirrors/vue-router/blob/dev/src/router.js#L88) 开始，里面调用 [```history.transitionTo()```](https://gitee.com/mirrors/vue-router/blob/dev/src/router.js#L135), ```setupListeners``` 是作为参数传进去的

```javascript

export default class VueRouter {
    ...

    init (app: any /* Vue component instance */) {
        ...

        if (history instanceof HTML5History || history instanceof HashHistory) {
            ...

            const setupListeners = routeOrError => {
                // 这个就是 setupListeners 
                // 外面包了一层
                history.setupListeners()
                // @TODO 这里好像关于滚动到页面锚点的东西
                handleInitialScroll(routeOrError)
            }
            // transitionTo 是从 History 类继承过来的
            history.transitionTo(
                history.getCurrentLocation(),
                setupListeners,
                setupListeners
            )
        }

        ...
    }

    ...

}

```

[```transitionTo```](https://gitee.com/mirrors/vue-router/blob/dev/src/history/base.js#L82) 把 ```setupListeners``` 包了一层之后继续向下传， 传到 [```confirmTransition```](https://gitee.com/mirrors/vue-router/blob/dev/src/history/base.js#L137) 中


[```transitionTo```](https://gitee.com/mirrors/vue-router/blob/dev/src/history/base.js#L82) 是从 [```History```](https://gitee.com/mirrors/vue-router/blob/dev/src/history/base.js#L25) 继承过来的

```javascript

export class History {
    ...

    // onComplete 就是传进来的 setupListeners
    transitionTo (location, onComplete, onAbort) {
        ...

        // 把 setupListeners(onComplete) 又作为第二个参数传进了这个方法里
        this.confirmTransition(
            route,
            () => {
                ...

                // 在这里调用
                onComplete && onComplete(route)

                // 这里还会执行导航守卫的钩子函数
                ...
            },
            err => {
                ...
            }
        )

        ...
    }

    // 这里的 onComplete 是在上面 transitionTo 里传入的第二个参数
    // setupListeners 就被包在里面
    // onComplete 执行时，setupListeners 就会被执行
    // 也就会添加 popstate 或 hashchange 的事件监听
    // 但是 onComplete 执行的位置很复杂
    // 是在传入 runQueue 的第三个参数中被执行调用了
    // 关于 runQueue 函数，可以看下面的解析
    // 主要就是一个类似迭代器的东西，第三个参数会在迭代完成的时候被调用
    confirmTransition (route, onComplete, onAbort) {
        ...

        runQueue(queue, iterator, () => {
            runQueue(queue, iterator, () => {
                ...
                // 在这里被调用了
                // 从 init 一路走到这里才开始监听
                onComplete(route)

                ...
            })
        })


        ...
    }

    ...
}

```

[```runQueue```](https://gitee.com/mirrors/vue-router/blob/dev/src/util/async.js#L3) 的代码并不多

```javascript
// 接收三个参数
// cb在遍历完所有queue中的项后被调用
// 向 fn 中传入两个参数
// 一个是当前遍历的项，另一个是控制进行下一步的函数(next)
export function runQueue (queue, fn, cb) {
    const step = index => {
        if (index >= queue.length) {
            cb()
        } else {
            if (queue[index]) {
                fn(queue[index], () => {
                    step(index + 1)
                })
            } else {
                step(index + 1)
            }
        }
    }
    step(0)
}

```







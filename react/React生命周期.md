# 说一说对React生命周期的理解

在16.3版本后，删除了3个生命周期：

* ```componentWillMount```
* ```componentWillReceiveProps```
* ```componentWillUpdate```

变为了

* ```UNSAFE_componentWillMount```
* ```UNSAFE_componentWillReceiveProps```
* ```UNSAFE_componentWillUpdate```

并增加了三个新的生命周期：

1. ```static getDerivedStateFromProps```
2. ```getSnapshotBeforeUpdate```
3. ```static getDerivedStateFromError```

## 16.3版本前旧的生命周期为：

![旧的React生命周期](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e12b2e35c8444f19b795b27e38f4c149~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

1. 挂载阶段：

* ```constructor```
* ```componentWillMount```
* ```render```
* ```componentDidMount```

2. 更新阶段
 
* ```componentWillReceiveProps```
* ```shouldComponentUpdate```
* ```componentWillUpdate```
* ```render```
* ```componentDidUpdate```

3. 卸载阶段

* ```componentWillUnmount```

4. 错误处理阶段

* ```componentDidCatch```


## 新的生命周期：
![16.3以后的生命周期](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a7d8676f379d4d96bbf0ebd9a8528594~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

1. 挂载阶段
* ```constructor```
* ```static getDerivedStateFromProps```
* ```render```
* ```componentDidMount```

2. 更新阶段

* ```static getDerivedStateFromProps```
* ```shouldComponentUpdate```
* ```render```
* ```getSnapshotBeforeUpdate```
* ```componentDidUpdate```

3. 卸载阶段

* ```componentWillUnmount```

4. 错误处理阶段

* ```static getDerivedStateFromError```
* ```componentDidCatch```



## 父子组件生命周期的调用顺序

### 父子组件第一次渲染的时候：

* Parent ```constructor```
* Parent ```static getDerivedStateFromProps```
* Parent ```render```
* Child ```constructor```
* Child ```static getDerivedStateFromProps```
* Child ```render```
* Child ```componentDidMount```
* Parent ```componentDidMount```

### 自组件修改自身state：

* Child ```static getDerivedStateFromProps```
* Child ```shouldComponentUpdate```
* Child ```render```
* Child ```getSnapshotBeforeUpdate```
* Child ```componentDidUpdate```

### 修改父组件中传入自组件的Props

* Parent ```static getDerivedStateFromProps```
* Parent ```shouldComponentUpdate```
* Parent ```render```
* Child ```static getEdrivedStateFromProps```
* Child ```shouldComponentUpdate```
* Child ```render```
* Child ```getSnapshotBeforeUpdate```
* Parent ```getSnapshotBeforeUpdate```
* Child ```componentDidUpdate```
* Parent ```componentDidUpdate```

### 卸载自组件

* Parent ```static getDerivedStateFromProps```
* Parent ```shouldComponentUpdate```
* Parent ```render```
* Parent ```getSnapshotBeforeUpdate```
* Child ```componentWillUnmount```
* Parent ```componentDidUpdate```

### 重新挂载子组件

* Parent ```static getDerivedStateFromProps```
* Parent ```shouldComponentUpdate```
* Parent ```render```
* Child ```constructor```
* Child ```static getDerivedStateFromProps```
* Child ```render```
* Parent ```getSnapshotBeforeUpdate```
* Child ```componentDidMount```
* Parent ```componentDidUpdate```


# setState是同步还是异步的

在 react18 之前，setstate大部分都是异步的，比如在react的合成事件中、react的生命周期中等；

但是在setTimeout等宏任务中是同步的，在原生的DOM2级事件监听中也是同步的

在react18后，只有使用原生DOM2级事件绑定的函数中才是同步的

# fiber架构

# React事件机制

# React-Router原理

# react-redux是如何工作的？



# 为什么不建议在shouldComponentUpdate生命周期里使用深度相等检查？

# React18有哪些更新？





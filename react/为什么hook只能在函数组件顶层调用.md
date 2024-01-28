# 为什么hook只能在函数组件顶层调用


## 结论：

初始化函数组件的时候，每一个hook执行会有```next```字段连接到下一个hook；

在更新的时候，通过 ```next``` 字段依次找到下一个hook，然后取上一次的结果，也就是```memoizedState```字段的值，；

如果某一个hook没有执行，就会通过 ```next``` 取到没有执行的下一个hook的值，不同hook的值格式是不相同的，所以会引发报错


比如这种结构：**useState => useRef => useEffect**

假设```useRef```在```if```语句中，初始化的时候```if```语句为```true```，在更新的时候```if```语句为```false```；那么会变成 **useState => useEffect**

```useState```的取值是正确的，但是 ```useEffect``` 拿到的是 ```useRef``` 的值，读取不到 create、deps 等字段值会报错

这也是为什么react更新可以中断的原因，当一个hook执行完成后，如果有更高优先级的任务，会暂停执行，在可以继续执行的时候，通过next可以找到下一个hook，继续进行更新计算。


下面贴上代码

每一个hook在使用的时候, 第一次调用会走到 [```mountWorkInProgressHook```](https://gitee.com/mirrors/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js#L820) 方法里, 在这里进行初始化, 在更新的时候会走到[```updateWorkInProgressHook```](https://gitee.com/mirrors/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js#L841)方法里, 在这里进行更新操作



## 初始化的逻辑
```javascript
// 初始化hook的时候调用
function mountWorkInProgressHook() {
  // 创建一个hook对象
  var hook = {
    // 缓存上一次的执行结果
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    // 有副作用的 hook 会把副作用存到这里
    queue: null,
    // 指向下一个 hook
    next: null
  };

  //  workInProgressHook 是全局变量，初始化的时候是空
  // 如果是null说明是初始化，会把第一个hook赋值给 workInProgressHook
  if (workInProgressHook === null) {
    // This is the first hook in the list
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // 如果不为null，就把当前的hook赋值给上一个hook的next属性
    // 这样每个hook就按照代码写的时候的先后顺序连起来
    // Append to the end of the list
    workInProgressHook = workInProgressHook.next = hook;
  }

  /**
   * 例如下面这段代码
   * @param  {{ 忽略这行(这样写可以让下面的内容高亮)
   * 
   * function Index(){
   *    const [ number , setNumber ] = useState(0)
   *    const curRef  = useRef(null)
   *    useEffect(()=>{
   *        console.log(1)
   *    }, [])
   *    
   *    ...
   * }
   * 
   * 
   * 最后变成下面这种结构，被赋值给Index组件的
   * FiberNode的memoizedState属性 
   * 下面结构只列出了关键字段
   * {
   *    // useState
   *    memoizedState: 0,
   *    next: {
   *        // useRef
   *        memoizedState: null,
   *        next: {
   *            // useEffect
   *            memoizedState:  {
   *                create: () => { console.log(1) },
   *                deps: [],
   *                destroy: undefined,
   *                ...
   *            },
   *            next: null
   *        }
   *    }
   * }
   * 
   * 这里与Fiber相关，先不展开
   */

  return workInProgressHook;
}

```



## 下面是更新的逻辑

假设执行了```setNumber(1)```，触发了更新

```javascript
// 根据上述hook结构(useState => useRef => useEffect)
// 函数会被执行三次
function updateWorkInProgressHook() {
  var nextCurrentHook;
  if (currentHook === null) {
    // 第一次 currentHook 为 null
    // 第二次不走这里; 此时 currentHook = (useState => useRef => useEffect)
    // 第三次不走这里; 此时 currentHook = (useRef => useEffect)

    // fiber相关的先略过
    // currentlyRenderingFiber.alternate.memoizedState 记录了需要替换上去的hook
    // 同样也是(useState => useRef => useEffect)这种结构(update)
    var current = currentlyRenderingFiber.alternate;

    if (current !== null) {
      // 第一次 current = FiberNode 走到这里
      // nextCurrentHook = (useState => useRef => useEffect)
      nextCurrentHook = current.memoizedState;
    } else {
      // 第一次不走这里
      nextCurrentHook = null;
    }
  } else {
    // 第一次不走这里
    // 第二次走这里; nextCurrentHook = （useRef => useEffect)
    // 第三次走这里; nextCurrentHook = （useEffect)

    nextCurrentHook = currentHook.next;
  }

  var nextWorkInProgressHook;

  if (workInProgressHook === null) {
    // 第一次走这里 workInProgressHook 为空
    // 拿到当前的值(useState => useRef => useEffect)
    nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
    // 第二次不走这里
    // 第三次不走这里
  } else {
    // 第一次不走这里
    // 第二次走这里; nextWorkInProgressHook = (useRef => useEffect)
    // 第三次走这里; nextWorkInProgressHook = (useEffect)
    nextWorkInProgressHook = workInProgressHook.next;
  }

  if (nextWorkInProgressHook !== null) {
    // 第一次 nextWorkInProgressHook = (useState => useRef => useEffect) 不为空
    // 第二次 nextWorkInProgressHook = (useRef => useEffect)
    // 第三次 nextWorkInProgressHook = (useEffect)

    // 第一次 workInProgressHook = (useState => useRef => useEffect)
    // 第二次 workInProgressHook = (useRef => useEffect)
    // 第三次 workInProgressHook = (useEffect)
    // There's already a work-in-progress. Reuse it.
    workInProgressHook = nextWorkInProgressHook;
    // 第一次 nextWorkInProgressHook = (useRef => useEffect)
    // 第二次 nextWorkInProgressHook = (useEffect)
    // 第三次 nextWorkInProgressHook = null
    nextWorkInProgressHook = workInProgressHook.next;
    // 第一次 currentHook = (useState => useRef => useEffect)
    // 第二次 currentHook = (useRef => useEffect)
    // 第三次 currentHook = (useEffect)
    currentHook = nextCurrentHook;
  } else {
    // 第一次不走这里
    // 第二次不走这里
    // 第三次不走这里
    // Clone from the current hook.
    if (nextCurrentHook === null) {
      var currentFiber = currentlyRenderingFiber.alternate;
      if (currentFiber === null) {
        // This is the initial render. This branch is reached when the component
        // suspends, resumes, then renders an additional hook.
        var _newHook = {
          memoizedState: null,
          baseState: null,
          baseQueue: null,
          queue: null,
          next: null
        };
        nextCurrentHook = _newHook;
      } else {
        // This is an update. We should always have a current hook.
        throw new Error('Rendered more hooks than during the previous render.');
      }
    }

    currentHook = nextCurrentHook;
    var newHook = {
      memoizedState: currentHook.memoizedState,
      baseState: currentHook.baseState,
      baseQueue: currentHook.baseQueue,
      queue: currentHook.queue,
      next: null
    };

    if (workInProgressHook === null) {
      // This is the first hook in the list.
      currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
    } else {
      // Append to the end of the list.
      workInProgressHook = workInProgressHook.next = newHook;
    }
  }

  // workInProgressHook = h2
  // 第一次 workInProgressHook = 
  // (useState => useRef => useEffect)
  // 第二次 workInProgressHook = 
  // (useRef => useEffect)
  // 第三次 workInProgressHook = 
  // (useEffect)
  return workInProgressHook;
}
```



[```updateWorkInProgressHook```](https://gitee.com/mirrors/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js#L841)函数的代码










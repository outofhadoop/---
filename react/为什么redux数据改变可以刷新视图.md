# 为什么 redux 的数据改变可以刷新视图

redux 会把组件通过 HOC（高阶组件）的方式包装一层，把 redux 的数据作为 props 传入组件内。

下面是 redux 的部份源码(没有 ts 和 production 部份)：

```javascript
function connect(
  mapStateToProps, //
  mapDispatchToProps,
  mergeProps,
  {
    pure, // 好像没什么用了
    areStatesEqual = strictEqual, // 用 === 判断是否相等
    areOwnPropsEqual = shalloweEqual, // 判断props是不是没变
    areStatePropsEqual = shallowEqual,
    areMergedPropsEqual = shallowEqual,
    forwardRef = false, // 是否用了 ref
    context = ReactReduxContext, // 拿到context
  }
) {
  const Context = context;
  // 看不懂
  const initMapStateToProps = mapStateToPropsFactory(mapStateToProps);
  const initMapDispatchToProps = mapDispatchToPropsFactory(mapDispatchToProps);
  const initMergeProps = mergePropsFactory(mergeProps);
  // 这不就是 true 么
  const shouldHandleStateChanges = Boolean(mapStateToProps);

  // 接收一个react组件WrappedComponent, 并返回一个新的连接后的ConnectFunction组件
  const wrapWithConnect = (WrappedComponent) => {
    // 拿到组件名 属性：displayName 、name
    const wrappedComponentName =
      WrappedComponent.displayName || WrappedComponent.name || "Component";
    // 被包裹后的组件名就是加了一个 "Connect"的前缀
    const displayName = `Connect(${wrappedComponentName})`;

    //  包含一系列用于创建选择器（selector）的选项，用于从 Redux store 的状态中选择并计算组件所需的数据
    // 就是把上面的很多参数放到 selectorFactoryOptions 中
    const selectorFactoryOptions = {
      shouldHandleStateChanges,
      displayName,
      wrappedComponentName,
      WrappedComponent,
      initMapStateToProps,
      initMapDispatchToProps,
      initMergeProps,
      areStatesEqual,
      areStatePropsEqual,
      areOwnPropsEqual,
      areMergedPropsEqual,
    };

    // 连接后的组件？
    function ConnectFunction(props) {
      const [propsContext, reactReduxForwardedRef, wrapperProps] =
        React.useMemo(() => {
          // props里有这些东西？
          const { reactReduxForwardedRef, ...wrapperProps } = props;
          return [props.context, reactReduxForwardedRef, wrapperProps];
        }, [props]);

      // 拿到context，用户可能自己写一个context 实例
      const ContextToUse = React.useMemo(() => {
        return propsContext &&
          propsContext.Consumer &&
          isContextConsumer(<propsContext.Consumer />)
          ? propsContext
          : Context;
      }, [propsContext, Context]);
      //   这里应该是拿到了context
      const contextValue = React.useContext(ContextToUse);
      // 检查是否存在一个store，必须存在
      const didStoreComeFromProps = Boolean(props.store) &&
        Boolean(props.store!.getState) &&
        Boolean(props.store!.dispatch)
      const didStoreComeFromContext =
        Boolean(contextValue) && Boolean(contextValue!.store)

      // 拿到store
      const store = didStoreComeFromProps
        ? props.store!
        : contextValue!.store

      // getState
      const getServerState = didStoreComeFromContext
        ? contextValue.getServerState
        : store.getState

      // 从store中筛选出对应的数据
      const childPropsSelector = React.useMemo(() => {
        return defaultSelectorFactory(store.dispatch, selectorFactoryOptions)
      }, [store])

      // 监听 store 变化
      const [subscription, notifyNestedSubs] = React.useMemo(() => {
        if (!shouldHandleStateChanges) return NO_SUBSCRIPTION_ARRAY

        // This Subscription's source should match where store came from: props vs. context. A component
        // connected to the store via props shouldn't use subscription from context, or vice versa.
        const subscription = createSubscription(
          store,
          didStoreComeFromProps ? undefined : contextValue!.subscription
        )
        // 这样做是为了确保 notifyNestedSubs 在后续使用时的上下文正确
        const notifyNestedSubs =
          subscription.notifyNestedSubs.bind(subscription)

        return [subscription, notifyNestedSubs]
      }, [store, didStoreComeFromProps, contextValue])



      // ？？？？
      const overriddenContextValue = React.useMemo(() => {
        if (didStoreComeFromProps) {
          return contextValue!
        }

        return {
          ...contextValue,
          subscription,
        } as ReactReduxContextValue
      }, [didStoreComeFromProps, contextValue, subscription])

      // ？？？？
      // 用于保存上一次从 store 中获取的子组件的 props。在组件的渲染周期中，
      // lastChildProps 会在每次渲染完成后更新为当前渲染得到的子组件的 props
      const lastChildProps = React.useRef()
      // 用于保存当前外部包裹组件的 props。由于 wrapperProps 是函数组件的参数，
      // 可能会在每次渲染中发生变化，
      // 而 lastWrapperProps 会在每次渲染完成后更新为当前的 wrapperProps
      const lastWrapperProps = React.useRef(wrapperProps)
      // 用于保存最近一次通过 store 更新后得到的子组件的 props。
      // 在 Redux 的 connect 函数中，当 store 发生变化时，
      // 会通过订阅机制通知所有连接的组件进行更新，这时会得到新的子组件的 props，
      // 并将其保存在 childPropsFromStoreUpdate 中
      const childPropsFromStoreUpdate = React.useRef()
      // 用于标记是否已经安排了组件的重新渲染。在 connect 函数中，
      // 会通过 notifyNestedSubs 函数安排组件的重新渲染，
      // 而 renderIsScheduled 可以帮助追踪是否已经有渲染操作被安排
      const renderIsScheduled = React.useRef(false)
      // 用于标记是否正在处理 Redux store 的 dispatch 操作。
      // 在 connect 函数中，为了避免多次触发渲染，
      // 通常会在 dispatch 操作期间暂时禁止重新渲染，
      // 而 isProcessingDispatch 可以帮助追踪是否正在处理 dispatch 操作
      const isProcessingDispatch = React.useRef(false)
      // 用于标记组件是否已经挂载。在 React 组件的生命周期中，
      // isMounted 会在组件挂载后更新为 true，在组件卸载后更新为 false
      const isMounted = React.useRef(false)
      // 应该是用来错误处理
      const latestSubscriptionCallbackError = React.useRef()


      // 判断组件是否已经挂载
      useIsomorphicLayoutEffect(() => {
        isMounted.current = true
        return () => {
          isMounted.current = false
        }
      }, [])




      /**
       * 它根据一些条件选择最终传递给子组件的属性。
       * 在 Redux 的架构中，当 Redux store 更新时，会触发组件重新渲染。
       * 这个选择器通过比较上一次包装组件属性 (wrapperProps) 是否与当前属性相同，
       * 来确定是否使用从 Redux store 更新中获取的子组件属性 (childPropsFromStoreUpdate)。
       * 这种机制的目的是尽可能减少不必要的渲染，只在真正需要更新子组件属性时才进行。
       * 选择器的结果将作为最终的子组件属性传递给 renderedWrappedComponent，从而影响组件的渲染结果
       */
      // 选择最终传递给子组件的属性
      const actualChildPropsSelector = React.useMemo(() => {
        const selector = () => {
          if (
            childPropsFromStoreUpdate.current &&
            wrapperProps === lastWrapperProps.current
          ) {
            return childPropsFromStoreUpdate.current
          }
          return childPropsSelector(store.getState(), wrapperProps)
        }
        return selector
      }, [store, wrapperProps])



      // 订阅 Redux store 的更新
      const subscribeForReact = React.useMemo(() => {
        const subscribe = (reactListener) => {
          if (!subscription) {
            return () => {}
          }
          // 处理订阅的更新逻辑
          // 在 store 更新时重新计算子组件属性，安排重新渲染等
          return subscribeUpdates(
            shouldHandleStateChanges,
            store,
            subscription,
            childPropsSelector,
            lastWrapperProps,
            lastChildProps,
            renderIsScheduled,
            isMounted,
            childPropsFromStoreUpdate,
            notifyNestedSubs,
            reactListener
          )
        }

        return subscribe
      }, [subscription])

      // useIsomorphicLayoutEffectWithArgs 在组件挂载和更新时执行
      useIsomorphicLayoutEffectWithArgs(captureWrapperProps, [
        lastWrapperProps,
        lastChildProps,
        renderIsScheduled,
        wrapperProps,
        childPropsFromStoreUpdate,
        notifyNestedSubs,
      ])

      let actualChildProps


      // 将 store 中的状态同步到组件中，以便在 store 更新时重新渲染组件
      try {
        actualChildProps = useSyncExternalStore(
          subscribeForReact,
          actualChildPropsSelector,
          getServerState
            ? () => childPropsSelector(getServerState(), wrapperProps)
            : actualChildPropsSelector
        )
      } catch (err) {
        ...
        throw err
      }

      useIsomorphicLayoutEffect(() => {
        // 清除上一次的订阅回调错误
        latestSubscriptionCallbackError.current = undefined
        // 清除上一次从 Redux store 更新而来的子组件 props
        childPropsFromStoreUpdate.current = undefined
        lastChildProps.current = actualChildProps
      })

      const renderedWrappedComponent = React.useMemo(() => {
        return (
          // 这就是传入的component
          // 将计算后的props和ref传入
          <WrappedComponent
            {...actualChildProps}
            ref={reactReduxForwardedRef}
          />
        )
      }, [reactReduxForwardedRef, WrappedComponent, actualChildProps])
      


      const renderedChild = React.useMemo(() => {
        if (shouldHandleStateChanges) {
          // 通过上下文传递了组件自己的订阅实例
          return (
            <ContextToUse.Provider value={overriddenContextValue}>
              {renderedWrappedComponent}
            </ContextToUse.Provider>
          )
        }

        return renderedWrappedComponent
      }, [ContextToUse, renderedWrappedComponent, overriddenContextValue])
      

      return renderedChild
    }

    const _Connect = React.memo(ConnectFunction)

    const Connect = _Connect

    Connect.WrappedComponent = WrappedComponent
    Connect.displayName = ConnectFunction.displayName = displayName
    if (forwardRef) {
      const _forwarded = React.forwardRef(function forwardConnectRef(
        props,
        ref
      ) {
        return <Connect {...props} reactReduxForwardedRef={ref} />
      })

      const forwarded = _forwarded as ConnectedWrapperComponent
      forwarded.displayName = displayName
      // 方便在 DevTools 中追踪组件的原始类型
      forwarded.WrappedComponent = WrappedComponent
      // 静态属性的复制和合并
      return hoistStatics(forwarded, WrappedComponent)
    }
    return hoistStatics(Connect, WrappedComponent)
  }
  return wrapWithConnect
}
```

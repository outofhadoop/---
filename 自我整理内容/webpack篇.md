# webpack 的构建流程？
* 合并options
* 初始化插件
* 初始化webpack
* 读取入口文件，并分析文件依赖
* 构建模块
* 输出编译后的文件

# webpack的tree shaking 原理？
[参考](https://juejin.cn/post/7002410645316436004)

# 插件的生命周期？
[参考](https://juejin.cn/post/7206487695123480635)

# loader和plugin的区别？

# 常见的loader和plugin，以及他们的作用？

# 优化打包速度的方式有哪些？

# 优化打包后体积的方式有哪些？

# webpack 的热更新原理？

webpack的 **webpack-dev-server** 会监听文件变化，通知webpack重新打包到内存中，打包完后会触发webpack的钩子事件 ```done```， webpack-dev-middleware 会监听这个事件，然后通过 websocket 发送一个```type```值为```hash``` 给浏览器，并带上变化的文件的hash值，然后会把这个值发给 webpack/hot/dev-server，判断是热更新还是立即刷新，如果需要刷新就立即刷新浏览器，如果是热更新，就把保存的hash通过ajax请求发送给webpack，检测确实有更新后，返回一个json格式的数据，包含所有的需要更新的文件列表，浏览器收到后，会通过websocket请求更新的代码，检查并删除过期的模块和依赖，然后替换新的代码。如果热更新过程出现错误，就立即刷新浏览器，回到之前的状态。

# 在使用 webpack 开发时，你用过哪些可以提供效率的插件？
# node 多线程

## [worker_threads](https://nodejs.org/dist/latest-v20.x/docs/api/worker_threads.html)

```javascript
const { Worker, parentPort } = require("worker_threads")
const wStr = `
    const { parentPort } = require("worker_threads")

    parentPort.postMessage('wStr');
    
    parentPort.on('message', (value) => 
        console.log('thread:' + value)
    )
`
const w = new Worker(wStr, {
    eval: true
})
w.on('message', (value) => {
    console.log(value)
})
w.postMessage('main: message')
```
### 1. 整体分析
Promise其实就是一个容器,有三个状态,分别是PENDING(进行中), FULFILLED(成功), REJECTED(失败),容器里面保存着未来才会结束事件的结果,它拥有两个特点:
- 容器里面的状态不受外部的影响
- 容器里得状态一旦改变就不会再变,任何时候都会得到这个结果
```javascript
new Promise((resolve, reject) => {
  // 成功执行resolve, 失败执行reject
}).then(res => {
  // 成功执行该方法
}, err => {
  // 失败执行该方法
}).then(res => {
  // 支持链式调用
}).catch(err => {

}).finally(res => {})

Promise.resolve()
Promise.reject()
Promise.race([promise1, promise2...]).then()
Promise.all([promise1, promise2...]).then()
```

根据上面的代码,我们可以得出
- Promise的构造函数接受一个callBack函数为参数,而callBack函数接受resolve和reject方法为参数,通过resolve和reject来控制状态(成功执行resolve, 失败执行reject)
- 状态改变之后触发原型链上的then和catch,finally方法
- 其中then方法含有两个回调函数参数
- Promise拥有静态方法all,race,resolve,reject

根据上面三条,我们可以得出一个整体大概的类如下:
```javascript
class _Promise{
    constructor(callBack){
        const resolve = ()=>{}
        const reject = ()=>{}
        callBack(resolve,reject)
    }
    then(){}
    catch(){}
    finally(){}
    static resolve(){}
    static reject(){}
    static race(){}
    static all(){}
}
```
### 2. 实现初版constructor
#### (1) 实现三个状态改变
我们知道resolve状态变成FULFILLED ,reject状态变为REJECTED,容器里得状态一旦改变就不会再变,任何时候都会得到这个结果,由于exector函数执行时可能会报错,所以我们需要try catch一下
```javascript
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

class _Promise{
    constructor(callBack){
        const PENDING = 'pending'
        const FULFILLED = 'fulfilled'
        const REJECTED = 'rejected'

        // 初始化
        this.status = PENDING
        this.res = undefined
        this.rej = undefined
        const resolve = (res)=>{
            if(this.status == PENDING){
                this.res = res
                this.status = FULFILLED
            }
        }
        const reject = (rej)=>{
            if(this.status == PENDING){
                this.rej = rej
                this.status = REJECTED
            }
        }
        try{
            // 立即执行executor
            // 把内部的resolve和reject传入executor，用户可调用resolve和reject
            callBack(resolve,reject)
        }catch(error){
            reject(error)
        }
        
    }
    then(){}
    catch(){}
    finally(){}
    static resolve(){}
    static reject(){}
    static race(){}
    static all(){}
}
```
#### (2) then的实现
then方法就是判断当前的状态,根据状态的不同执行不同的方法,由于then和catch是微任务,这里用settimeout代替
```javascript
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

class _Promise{
    constructor(callBack){
        // 初始化
        this.status = PENDING
        this.res = undefined
        this.rej = undefined
        const resolve = (res)=>{
            if(this.status == PENDING){
                this.res = res
                this.status = FULFILLED
            }
        }
        const reject = (rej)=>{
            if(this.status == PENDING){
                this.rej = rej
                this.status = REJECTED
            }
        }
        try{
            // 立即执行callBack
            // 把内部的resolve和reject传入callBack，用户可调用resolve和reject
            callBack(resolve,reject)
        }catch(error){
            reject(error)
        }
        
    }
    then(onFulfilled, onRejected){
        // then是微任务，这里用setTimeout模拟
        setTimeout(() => {
            if (this.status === FULFILLED) {
                // FULFILLED状态下才执行
                onFulfilled(this.res)
            } else if (this.status === REJECTED) {
                // REJECTED状态下才执行
                onRejected(this.rej)
            }
        });
    }
    catch(){}
    finally(){}
    static resolve(){}
    static reject(){}
    static race(){}
    static all(){}
}
```
测试上面函数
```javascript
new _Promise((resolve, reject) => {
  resolve(111)
}).then(res => {
  console.log(res)  // 111
})
```

### 3 then支持异步回调
```javascript
new Promise((resolve, reject) => {
  setTimeout(()=>{
    resolve(111)
  },0)
}).then(res => {
  console.log(res)  // 111
})
```
如果Promise内部有异步代码，并且等到异步执行完毕才调用resolve(reject)，而then函数不会等待执行完毕再执行，此时状态还是pending，所以不会执行回调函数,对此有两种方式解决
- 等待异步执行完毕再重新调用then
- 在调用then的时候，如果状态还是pending则将成功和失败的回调存储，当resolve(reject)执行完毕再调用

我们采用第二种思路，这时我们要引入两个变量onFulfilledCallbacks和onRejectedCallbacks,用来存储要执行的回调函数,在resolve和reject执行改变状态之后再执行回调函数,其实就是发布订阅模式
```javascript
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

class _Promise{
    constructor(cb){
        this.status = PENDING
        this.res = undefined
        this.rej = undefined

        this.onFulfilledCallbacks = []
        this.onRejectedCallbacks = []
        
        const resolve = (res)=>{
            if(this.status == PENDING){
                this.status = FULFILLED
                this.res = res
                // 成功态函数依次执行
                this.onFulfilledCallbacks.forEach((fn)=>{fn(this.res)})
            }
        }
        const reject = (rej)=>{
            if(this.status == PENDING){
                this.status = REJECTED
                this.rej = rej
                // 失败态函数依次执行
                this.onRejectedCallbacks.forEach((fn)=>{fn(this.rej)})
            }
        }

        try{
            cb(resolve,reject)
        }catch(err){
            reject(err)
        }
    }
    then(onFulfilled,onRejected){
        setTimeout(()=>{
            if(this.status == PENDING){
                // 状态是PENDING下执行
                // 说明promise内部有异步代码执行，还未改变状态，添加到成功/失败回调任务队列即可
                this.onFulfilledCallbacks.push(onFulfilled)
                this.onRejectedCallbacks.push(onRejected)
            }else if(this.status == FULFILLED){
                onFulfilled(this.res)
            }else if(this.status == REJECTED){
                onRejected(this.rej)
            }
        },0)
    }
}
```

### 4. then链式调用
##### (1) then的返回值为非promise
示例1
```javascript
const pro = new Promise((resolve,reject)=>{
    resolve('then1')
}).then((res)=>{
    console.log(res)
    return 'then2'
})
setTimeout(()=>{
    console.log(pro)
},0)
```

输出如下
```
then1
Promise {<fulfilled>: then2}
```

示例2
```javascript
const pro = new Promise((resolve,reject)=>{
    reject('then1')
}).then((res)=>{
    console.log(res)
    return new Error('Error1')
},(err)=>{
    console.log(err,'err1')
    return new Error('Error2')
})
setTimeout(()=>{
    console.log(pro)
},0)
```

输出如下
```
then1 err1
Promise {<fulfilled>: Error: Error2
    at <anonymous>:8:12}
```

示例3
```javascript
const pro = new Promise((resolve,reject)=>{
    reject('then1')
}).then((res)=>{
    console.log(res)
    return new Error('Error1')
},(err)=>{
    console.log(err,'err1')
    throw 'Error'
})
setTimeout(()=>{
    console.log(pro)
},0)
```

输出如下
```
then1 err1
Promise {<rejected>: 'Error'}
```

示例4
```javascript
const pro = new Promise((resolve,reject)=>{
    resolve('then1')
}).then((res)=>{
    console.log(res)
    return 'then2'
}).then((res)=>{
    console.log(res)
})
setTimeout(()=>{
    console.log(pro)
},0)
```

输出如下
```
then1
then2
Promise {<fulfilled>: undefined}
```


示例5
```javascript
const pro = new Promise((resolve,reject)=>{
    resolve('then1')
}).then((res)=>{
    console.log(res)
    return 'then2'
}).then((res)=>{
    console.log(res)
    throw 'Error'
},(err)=>{
    console.log(err,'err1')
}).catch((err)=>{
    console.log(err,'err2')
})
setTimeout(()=>{
    console.log(pro)
},0)
```

输出如下
```
then1
then2
Error err2
Promise {<fulfilled>: undefined}
```

输出结果分析
- then每次都能返回一个新的promise,状态为fulfilled
- then返回新的promise值和状态根据上一个then返回，如无返回则为undefined，状态根据目前情况直接fulfilled或者reject(注意:如果是在then返回异常数据,例如Error,状态也为fulfilled,而在then抛出异常会返回rejected)
- then抛出异常后会被catch或者后续的then的第二个回调接收

```javascript
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

class _Promise{
    constructor(cb){
        this.status = PENDING
        this.res = undefined
        this.rej = undefined

        this.onFulfilledCallbacks = []
        this.onRejectedCallbacks = []

        const resolve = (res)=>{
            if(this.status == PENDING){
                this.status = FULFILLED
                this.res = res
                this.onFulfilledCallbacks.forEach((fn)=>{
                    fn(this.res)
                })
            }
        }

        const reject = (rej)=>{
            if(this.status == PENDING){
                this.status = REJECTED
                this.rej = rej
                this.onRejectedCallbacks.forEach((fn)=>{
                    fn(this.res)
                })
            }
        }

        try{
            cb(resolve,reject)
        }catch(err){
            reject(err)
        }
    }
    then(onFulfilled,onRejected){
        const self = this
        return new _Promise((resolve,reject)=>{
            if(self.status == PENDING){
                // 状态是PENDING下执行
                // 说明promise内部有异步代码执行，还未改变状态，添加到成功/失败回调任务队列即可
                self.onFulfilledCallbacks.push(()=>{
                       try{
                            setTimeout(()=>{
                                resolve(onFulfilled(self.res))
                            },0)
                       }catch(err){
                            reject(err)
                       } 
                })
                
                self.onRejectedCallbacks.push(()=>{
                       try{
                            setTimeout(()=>{
                                resolve(onRejected(self.rej))
                            },0)
                       }catch(err){
                            reject(err)
                       } 
                })
            }else if(self.status == FULFILLED){
                setTimeout(() => {
                    try {
                        resolve(onFulfilled(self.res))
                    } catch (error) {
                        reject(error)
                    }
                });
            }else if(self.status == REJECTED){
                setTimeout(() => {
                    try {
                        resolve(onRejected(self.rej))
                    } catch (error) {
                        reject(error)
                    }
                });
            }
            
        })
    }
}
```
##### (2) then的返回值为promise
示例1
```javascript
const pro = new Promise((resolve,reject)=>{
    resolve('then1')
}).then((res)=>{
    return new Promise((resolve,reject)=>{
        resolve('then2')
    })
})
setTimeout(()=>{
    console.log(pro)
},0)
```

输出如下
```
Promise {<fulfilled>: 'then2'}
```

示例2
```javascript
const pro = new Promise((resolve,reject)=>{
    resolve(new Promise((resolve,reject)=>{
        resolve('then1')
    }))
}).then((res)=>{
    return new Promise((resolve,reject)=>{
        console.log(res)
        resolve('then2')
    })
})
setTimeout(()=>{
    console.log(pro)
},0)
```

输出如下
```
then1
Promise {<fulfilled>: 'then2'}
```

示例3
```javascript
const pro = new Promise((resolve,reject)=>{
    resolve('then1')
}).then((res)=>{
    return new Promise((resolve,reject)=>{
        resolve('then2')
    })
}).then((res)=>{
    console.log(res)
})
setTimeout(()=>{
    console.log(pro)
},0)
```

输出如下
```
then2
Promise {<fulfilled>: undefined}
```

输出结果分析
- 返回值是一个promise实例,那么返回的下一个promise实例会等待这个promise状态发生变化

```javascript
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

class _Promise{
    constructor(cb){
        this.status = PENDING
        this.res = undefined
        this.rej = undefined

        this.onFulfilledCallbacks = []
        this.onRejectedCallbacks = []
        const resolve = (res)=>{
            if(this.status == PENDING){
                this.status = FULFILLED
                this.res = res
                this.onFulfilledCallbacks.forEach((fn)=>{
                    fn(this.res)
                })
            }
        }
        const reject = (rej)=>{
            if(this.status == PENDING){
                this.status = REJECTED
                this.rej = rej
                this.onRejectedCallbacks.forEach((fn)=>{
                    fn(this.rej)
                })
            }
        }
        try{
            cb(resolve,reject)
        }catch(error){
            reject(error)
        }
    }
}
then((onFulfilled,onRejected)=>{
    const self = this;
    return new _Promise((resolve,reject)=>{
        if(self.status == PENDING){
            self.onFilfulledCallbacks.push(()=>{
                try{
                    setTimeout(()=>{
                        // 分两种情况：
                        // 1. 回调函数返回值是Promise，执行then操作
                        // 2. 如果不是Promise，调用新Promise的resolve函数
                        const result = onFulfilled(self.res)
                        result instanceof _Promise ? result.then(resolve, reject) : resolve(result)
                    })
                }catch(error){
                    reject(error)
                }
            })

            self.onRejectedCallbacks.push(() => {
                // try捕获错误
                try {
                    // 模拟微任务
                    setTimeout(() => {
                        const result = onRejected(self.reason)
                        // 分两种情况：
                        // 1. 回调函数返回值是Promise，执行then操作
                        // 2. 如果不是Promise，调用新Promise的resolve函数
                        result instanceof _Promise ? result.then(resolve, reject) : resolve(result)
                    });
                } catch (error) {
                    reject(error)
                }
            })
        }else if(self.status == FULFILLED){
            setTimeout(() => {
                try {
                    // 分两种情况：
                    // 1. 回调函数返回值是Promise，执行then操作
                    // 2. 如果不是Promise，调用新Promise的resolve函数
                    result instanceof _Promise ? result.then(resolve, reject) : resolve(result)
                } catch (error) {
                    reject(error)
                }
            });
        }else if(self.status == REJECTED){
            setTimeout(() => {
                try {
                    // 分两种情况：
                    // 1. 回调函数返回值是Promise，执行then操作
                    // 2. 如果不是Promise，调用新Promise的resolve函数
                    result instanceof _Promise ? result.then(resolve, reject) : resolve(result)
                } catch (error) {
                    reject(error)
                }
            });
        }
    })
})
```
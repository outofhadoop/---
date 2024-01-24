### 1、三者的区别

1）三者都可以显式绑定函数的this指向
2）三者第一个参数都是this要指向的对象，若该参数为undefined或null，this则默认指向全局window
3）传参不同：apply是数组、call是参数列表，而bind可以分为多次传入，实现参数的合并
4）call、apply是立即执行，bind是返回绑定this之后的函数，如果这个新的函数作为构造函数被调用，那么this不再指向传入给bind的第一个参数，而是指向新生成的对象

### 2、call

call 函数的实现步骤：
1. 判断调用对象是否为函数
2. 判断传入的上下文对象是否存在，如果不存则设置为window
3. 将函数作为上下文对象的一个方法属性。
4. 使用隐式绑定改变调用函数this的指向，并且执行调用函数
5. 删除刚才新增的方法
6. 返回结果

```javascript
Function.prototype.myCall = function(obj,...args){
    // 判断调用对象
    if (typeof this !== "function") {
        throw new TypeError("Error");
    }
    // obj为undefined或null时，则this默认指向全局window
    obj = obj ? obj : window
    // 利用Symbol创建一个唯一的key值，防止新增加的属性与obj中的属性名重复
    let fn = Symbol()
    // this指向调用call的函数，将函数设为对象的方法
    obj[fn] = this
    // 隐式绑定this，如执行obj.foo(), foo内的this指向obj
    let res = obj[fn](...args);
    // 执行完以后，删除新增加的属性
    delete obj[fn]; 
    return res;
}

复制代码
```

### 3、apply

apply 函数的实现步骤：
1. 判断调用对象是否为函数
2. 判断传入的上下文对象是否存在，如果不存则设置为window
3. 将函数作为上下文对象的一个方法属性。
4. 使用隐式绑定改变调用函数this的指向，并且执行调用函数
5. 删除刚才新增的方法
6. 返回结果

```javascript
Function.prototype.myApply = function(obj,args){//apply传入是数组,call是多个参数
    // 判断调用对象
    if (typeof this !== "function") {
        throw new TypeError("Error");
    }
    // obj为undefined或null时，则this默认指向全局window
    obj = obj ? obj : window
    // 利用Symbol创建一个唯一的key值，防止新增加的属性与obj中的属性名重复
    let fn = Symbol()
    // this指向调用call的函数，将函数设为对象的方法
    obj[fn] = this // ==>>等同于obj.fn = this
    // 隐式绑定this，如执行obj.foo(), foo内的this指向obj
    let res = obj[fn](...args);
    // 执行完以后，删除新增加的属性
    delete obj[fn]; 
    return res;
}

复制代码
```

### 4、bind
bind 函数的实现步骤：
1. 判断调用对象是否为函数
2. 判断传入的上下文对象是否存在，如果不存则设置为window
3. 判断是否为构造函数new出来的实例
4. 利用闭包保存绑定值

```javascript
Function.prototype.myBind = function(obj,...args){
    // 判断调用对象
    if (typeof this !== "function") {
        throw new TypeError("Error");
    }
    // 判断上下文对象是否存在
    obj = obj ? obj : window
    let fn = this;
    let f = Symbol();
    const result = function(...args1) {
        if (this instanceof fn) {
            // result如果作为构造函数被调用，this指向的是new出来的对象
            // this instanceof fn，判断new出来的对象是否为fn的实例
            this[f] = fn;
            let res = this[f](...args, ...args1);
            delete this[f];
            return res;
        } else {
            // bind返回的函数作为普通函数被调用时
            obj[f] = fn;
            let res = obj[f](...args, ...args1);
            delete obj[f];
            return res;
        }
    };
    // 如果绑定的是构造函数 那么需要继承构造函数原型属性和方法
    // 实现继承的方式: 使用Object.create
    result.prototype = Object.create(fn.prototype);
    return result;
}

复制代码
```

### 5、new

1. 创建空对象
2. 设置原型，将对象的原型设置为函数的 prototype 对象。（也就是将对象的__proto__属性指向构造函数的prototype属性）
3. 让函数的 this 指向这个对象，执行构造函数的代码（为这个新对象添加属性）
4. 判断函数的返回值类型，如果是值类型，返回创建的对象。如果是引用类型，就返回这个引用类型的对象。

```javascript
function Person (name) {
  this.name = name
  return 1
}
Person.prototype.eat = function () {
  console.log('Eatting')
}
var person1 = new Person('person1')
console.log(person1) // Person {name: 'person1'} 函数返回的是非对象类型，所以返回创建对象
person1.eat() // Eatting

function create (Fun,...args) {
  // 1. 创建一个新对象
  let obj = {}
  // 2. 将新对象的_proto_指向构造函数的prototype对象
  obj._proto_ = Fun.prototype 
  // 3. 将构造函数的作用域赋值给新对象 （也就是this指向新对象）
  let res = Fun.apply(obj, args)
  // 4. 如果该函数没有返回对象，则返回this,优先返回构造函数返回的对象
  return res instanceof Object ? res : obj;
}

复制代码
```

### 6、instanceof

1. 特殊点，instanceof判断方法只能左边是实例对象(person)，右边是构造函数(Person)，所以可以通过获取参数一对象的原型(__prtoo__)，获取参数二的prototype方法进行判断。
2. 获取对象的原型，Object.getPrototypeOf() 方法，可以通过这个方法来获取对象的原型，即__proto__属性。
3. 获取函数的prototype对象
4. 然后一直循环(递归)判断对象的原型是否等于类型的原型，直到对象原型为 `null`，因为原型链最终为 `null`


```javascript
// 特殊的Object、Function
console.log(Object instanceof Object); //true
console.log(Function instanceof Function); //true
console.log(Function instanceof Object); //true
console.log(function() {} instanceof Function); //true

function instanceOf(person,Person){
    let proto = Object.getPrototypeOf(person)，
    prototype = Person.prototype;

    if(proto){
        if(proto === prototype){
            return true
        }else{
            instanceOf(proto,Person)
        }
    }else{
        return false
    }
}

// 测试
function Dog() {}
let dog = new Dog();
console.log(instanceOf(dog, Dog), instanceOf(dog, Object)); // true true

复制代码
```
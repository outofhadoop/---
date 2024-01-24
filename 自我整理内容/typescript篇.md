# interface和type有什么异同

interface 和 type 相同点： 

* 可以声明一个对象类型，并可以拓展这个类型的属性
* 编译后，这些声明不会存在，不像class，虽然可以定义对象类型，但是在编译后，class声明还是会在代码中存在

interface和type的不同点： 

* interface可以继承（继承interface或type或者是class声明的类型），但是type 是不能继承的，type拓展属性的方式是使用交叉类型
* type可以声明其他的类型，而interface只能声明对象类型
* interface有接口合并的特性，同名的interface中不同名属性可以自动合并，但是type是不可以的，同名的type声明会报错
* interface 中可以使用```this```关键字， 但是type中是不可以的
* interface 无法拓展原始数据类型，type可以做到: ```type Mystr =  string & {name: string}```
* interface 无法表达某些复杂类型（比如交叉类型和联合类型），但是type可以: ```type A = {a: string}; type B = {b: string}; type C = A | B ; type CName = C & {name: string};```

# TypeScript 中 const 和 readonly 的区别？
```const```用于声明一个地址不可变的变量，一般这个变量内部的属性是可以改变的，如果需要声明内部属性也是不可变的，需要使用```readonly```
```readonly```用于表示一个属性或者数组的内容是不可变的，只读的


# 枚举和常量枚举的区别？

| 特性                   | 普通枚举（Enum）                             | 常量枚举（Const Enum）                      |
|------------------------|-----------------------------------------------|--------------------------------------------|
| 存在形式               | 运行时存在，生成真实对象                     | 编译时被删除，内联表达式                   |
| 访问方式               | 可以在运行时通过枚举名或枚举值访问           | 编译后会被直接替换为相应的常量表达式      |
| 反向映射               | 支持从枚举值到枚举名的反向映射               | 不支持从枚举值到枚举名的反向映射            |
| 成员初始化             | 可以包含计算值或者被初始化为字符串、数字等   | 必须具有常量表达式，不能包含计算值         |
| 代码体积               | 生成的代码相对较大                           | 编译后的代码体积较小                       |
| 适用场景               | 需要在运行时使用枚举值或反向映射的情况       | 不需要在运行时访问枚举值，且希望减小代码体积 |



# TypeScript 中的 this 和 JavaScript 中的 this 有什么差异？

TypeScript中可以显示的指定this的类型，在调用函数的时候，可以避免js中this指向不明确的问题

# TypeScript 中使用 Union Types 时有哪些注意事项？
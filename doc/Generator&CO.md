# ES6 Generator & CO

koa 和 egg 是基于 CO 实现，CO 使用了 ES6 Generator 的特性。强烈建议先学习下阮一峰的 [ES6 Generator](http://es6.ruanyifeng.com/#docs/generator) 相关知识。

### ES6 Generator

先对 ES6 Generator 知识做个总结（可跳过）：

* 是一个状态机，同时还会返回遍历器对象（代表函数内部指针，value 表示当前状态值，done 表示遍历是否结束）

  ```{ value：''， done: boolean }```

* 调用 generator 函数后，该函数并不执行，返回的是遍历器对象，需要启用 next 方法将指针移向下一个状态

* return，如果没有 return 则遍历器对象的 value 为 undefined

* 惰性加载，只有 next 方法执行当前 yield 语句时，才会对当前 yield 语句进行求值

* yield 语句默认返回 undefined，但是 next 方法可以带一个参数作为**上一个** yield 语句的返回值（但是第一个 next 方法不能带有参数，会被 V8 引擎忽略）

* 使用 ```for … of``` 循环 generator 时， 一旦 next 方法返回的对象的 done 属性为 ture，循环就会终止，且不包含该返回对象（例如 return 的值）

* generator.prototype.return()，return 方法不提供参数时，返回值的 value 为 undefined，提供参数时 value 为对应的参数，done 的属性为 true，遍历终止（遇到 try finally 例外，会执行完 finally 的代码块再终止）

* 使用 ```yield foo();``` 时默认返回一个遍历器对象，使用 ```yield* foo```返回遍历器对象的值，这样可以在 generator 函数内部调用另一个 generator 函数

* 可以使用遍历器对象的 throw 方法，在函数体外抛出错误，然后在 Generator 函数体内捕获

* 遍历器对象如何获取正常的 this 对象实例？

  * 先生成一个空对象，然后使用 call 方法绑定 Generator 函数内部的 this

    ```var obj = {};```

    ```var f = F.call(obj);```

  * 另一种方法是使用遍历器对象的原型

    ```var f = F.call(F.prototype);```

那么，Generator 有什么作用？能解决什么问题呢？

* 状态机，Generator 是实现状态机的最佳结构，在几种状态间切换
* 解决回调地狱问题

### Generator 如何解决自动执行问题

首先，我们怎么样按次序**自动执行**一个 Generation 函数？

```Js
function* run(value1) {
  var value2 = yield step1(value1);
  var value3 = yield step2(value2);
  var value4 = yield step3(value3);
  // ...
}
```

做法如下：

```js
var g = run(initValue);
var res = g.next();

while (!res.done) {
  var result = res.value;
  res = g.next(result);
}
```

通过循环判断遍历器对象的 done 属性，不过这种做法只是在同步状态下，以异步下就没有这么简单了，以一个读取文件```package.json```为例：

```js
var fs = require('fs');

var callback = function (err, data) {
  console.log(data);
  // ...
}

var gen = function* () {
  var r1 = yield fs.readFile('./package.json', 'utf-8', function (err, data) {
    callback(err, data);
  });
  console.log(r1.toString());

  var r2 = yield fs.readFile('./package.json', 'utf-8', function (err, data) {
    callback(err, data);
  });
  console.log(r2.toString());
}

var g = gen();

var r1 = g.next();

console.log(r1);
// { value: undefined, done: false }
```

可以看到，```r1.value``` 为 undefined，无法正常输出 ```package.json``` 的内容。原因是无法知道文件何时读取完。因为无法在 generator 中的 fs.readFile 异步函数里面直接输出 data，解决的思路是将 callback 输出给 ```g.next().value```，也即是 ```yield { value: callback(), done: boolean }```，代码如下：

```js
var readFileThunk = function () { ... };

var callback = function (err, data) {
  console.log(data);
}

var gen = function* () {
  var r1 = yield readFileThunk('./package.json', 'utf-8');
  console.log(r1.toString());

  var r2 = yield readFileThunk('./package.json', 'utf-8');
  console.log(r2.toString());
}

var g = gen();

var r1 = g.next();

console.log(r1.value);

r1.value(callback());
```

readFileThunk 如何实现呢，它要实现以下两个功能，

* 处理参数
* 返回 callback 函数，并输出给遍历器对象的 value

这就是接下来要说的 Thunk 函数。

### Thunk 函数

> 一种自动执行 Generator 函数的一种方法

* 传值调用：在进入函数体前，计算表达式的值
* 传名调用：将表达式传入函数体，只在用到它时求值

在编译器中，Thunk 函数就是传名调用的一种实现策略，用来替换某个表达式。但是在 JavaScript 语言中 是传值调用，Thunk 函数替换的不是表达式，而是多参数函数，将其替换成一个**只接受回调函数作为参数的单参数函数**。可以理解成一个转换函数。

根据上面的例子，我们需要使用 Thunk 函数来使一个多参数函数（fs.readFile）替换成一个单参数函数且该参数必须为回调函数（function(callback)）。实现如下：

```js
var Thunk = function (fn) {
  return function () {
    var args = Array.prototype.slice.call(arguments);
    return function (callback) {
      args.push(callback);
      return fn.apply(this, args);
    }
  }
}

var readFileThunk = Thunk(fs.readFile);
```

根据上面的例子，解决了异步请求何时完成的问题，还有一个问题是如何自动执行的问题，其实 Thunk 函数同时也解决了。

```js
var g = gen();

var r1 = g.next();

r1.value(function (err, data) {
  if (err) throw err;
  var r2 = g.next(data);
  r2.value(function (err, data) {
    if (err) throw err;
    g.next(data);
  });
});
```

仔细查看上面的代码，可以发现 Generator 的执行过程，其实是将同一个回调函数反复传入 next 方法的 value 属性，这使得我们可以用递归来自动完成这个过程。

[Thunkify](https://github.com/tj/node-thunkify) 模块就是 Thunk 函数的一种实现，这里就不说明了。

### CO 框架

> 用于 Generator 函数的自动执行

除了回调函数可以使异步操作了有结果后才自己交加执行权，还有 Promise 对象也可以。CO 就是将这两种（Thunk 函数和 Promise 对象）包装成一个模块。

co 的使用非常简单，但是注意 ```yield``` 命令后面，只能是 Thunk 函数或者 Promise 对象，如果数组或对象的成员，全部都是 Promise 对象，也可以使用 co：

```js
var co = require('co');

co(gen).then(function () {
  console.log('执行完成');
});
```

 co 的原理主要是下面这几行代码：

```js
onFulfilled();
function onFulfilled(res) {
  var ret;
  try {
    ret = gen.next(res);
  } catch (e) {
    return reject(e);
  }
  next(ret);
}

function next(ret) {
  if (ret.done) return resolve(ret.value);
  var value = toPromise.call(ctx, ret.value);
  if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
  return onRejected(
    new TypeError(
      'You may only yield a function, promise, generator, array, or object, '
      + 'but the following object was passed: "'
      + String(ret.value)
      + '"'
    )
  );
}
```

* 使用 ```onFulfilled``` 方法执行 ```gen.next```，有异常则终止，没有异步调用 next()
* next 是最关键的函数，它会反复调用自身
  * 第一行，检查遍历是否完成，是就返回
  * 第二行，确保每一步的返回值是 Promise 对象
  * 第三行，使用 then 方法，通过 ```onFulfilled``` 方法再次调用 next 函数
  * 第四行，在参数不符合要求的情况下（参数非 Thunk 函数和 Promise 对象），将 Promise 对象的状态改为`rejected`，从而终止执行

### 学习资料

* [阮一峰 ES6 Generator](http://es6.ruanyifeng.com/#docs/generator)
* [Co](https://github.com/tj/co)

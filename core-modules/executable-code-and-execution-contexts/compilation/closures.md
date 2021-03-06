# 闭包

在了解闭包之前，先要熟悉以下几点： 

1. 首先要理解执行环境（[执行上下文栈](../execution/execution-context-stack.md)），执行环境定义了变量或函数有权访问的其他数据。
2. 每个执行环境都有一个与之关联的 [变量对象](../execution/variable-object.md)，环境中定义的所有变量和函数都保存在这个对象中。
3. 每个函数都有自己的执行环境，当执行流进入一个函数时，函数的环境就会被推入到一个环境栈中。而在函数执行之后，栈将其环境弹出，把控制权返回给之前的执行环境。
4. 当某个函数被调用时，会创建一个执行环境及其相应的 **作用域链**。然后使用 `arguments` 和其他命名参数的值来初始化函数的活动对象。在函数中，活动对象作为变量对象使用（*作用域链是由每层的变量对象链起来的*）。
5. 在作用域链中，外部函数的活动对象始终处于第二位，外部函数的外部函数的活动对象处于第三位，直到作用域链终点即全局执行环境。
6. **作用域链的本质是一个指向变量对象的指针列表，它只引用但不实际包含变量对象。**

## 定义

MDN 对闭包的定义为：

> 闭包是指那些能够访问自由变量的函数。

那什么是自由变量呢？

> 自由变量是指在函数中使用的，但既不是函数参数也不是函数的局部变量的变量。

由此，我们可以看出闭包共有两部分组成：

> 闭包 = 函数 + 函数能够访问的自由变量

🌰 **标准示例：**

```js
const a = 1;

function foo() {
    console.log(a);
}

foo();
```

`foo` 函数可以访问变量 `a`，但是 `a` 既不是 `foo` 函数的局部变量，也不是 `foo` 函数的参数，所以 `a` 就是自由变量。

那么，函数 `foo` 和函数访问的自由变量 `a` 就构成闭包。但是这只是**理论上**的闭包。

🎉 在 ECMAScript 中，闭包指的是：

* 从理论角度：所有的函数。因为它们都在创建的时候就将上层上下文的数据保存起来了。哪怕是简单的全局变量也是如此，因为函数中访问全局变量就相当于是在访问自由变量，这个时候使用最外层的作用域。

* 从实践角度：以下函数才算是闭包
  * 即使创建它的执行上下文已经销毁，它仍然存在（比如，内部函数从父函数中返回）
  * 在代码中引用了自由变量

## 分析

我们通过一段代码仔细分析执行过程到底发生了什么：

```js
function foo(){
    var a = 2;

    function bar(){
        console.log(a);
    }

    return bar;
}

var baz = foo();

baz();
```

1. 代码执行流进入全局执行环境，并对全局执行环境中的代码进行声明提升。
2. 执行流执行 `var baz = foo()` ，调用 `foo()` 函数，此时执行流进入 `foo()` 执行环境中，对该执行环境中的代码进行声明提升过程。此时执行环境栈中存在两个执行环境，`foo()` 函数为当前执行流所在执行环境。
3. 执行流执行代码 `var a = 2;`，对 `a` 进行 LHS 查询，给 `a` 赋值 2。
4. 执行流执行 `return bar` ，将 `bar()` 函数作为返回值返回。按理说，这时 `foo()` 函数已经执行完毕，应该销毁其执行环境，等待垃圾回收。但因为其返回值是 `bar` 函数。`bar` 函数中存在自由变量 `a`，需要通过作用域链到 `foo()` 函数的执行环境中找到变量 `a` 的值，所以虽然 `foo` 函数的执行环境被销毁，但其变量对象不能被销毁，**只是从活动状态变成非活动状态**；而全局环境的变量对象则变成活动状态；执行流继续执行 `var baz = foo()`，把 `foo()` 函数的返回值 `bar` 函数赋值给 `baz`。
5. 执行流执行 `baz()` ，通过在全局执行环境中查找 `baz` 的值，`baz` 保存着 `foo()` 函数的返回值 `bar`。所以这时执行 `baz()` ，会调用 `bar()` 函数，此时执行流进入 `bar()` 函数执行环境中，对该执行环境中的代码进行声明提升过程。此时执行环境栈中存在三个执行环境，`bar()` 函数为当前执行流所在的执行环境。
6. 在声明提升的过程中，由于 `a` 是个自由变量，需要通过 `bar()` 函数的作用域链 `bar() -> foo() -> 全局作用域` 进行查找，最终在 `foo()` 函数中找到 `var a = 2;` ，然后在 `foo()` 函数的执行环境中找到 `a` 的值是 2，所以给 `a` 赋值 2。
7. 执行流执行 `console.log(a)` ，调用内部对象 `console`，并从 `console` 对象中找到 `log` 方法，将 `a` 作为参数传递进去。从 `bar()` 函数的执行环境中找到 `a` 的值是 2，所以，最终在控制台显示 2。
8. 执行流执行完 ` bar()` 函数后，`bar()` 的执行环境被弹出执行环境栈，并被销毁，等待垃圾回收，控制权还给全局执行环境。
9. 当页面关闭时，所有执行环境都被销毁。

```js
// 执行上下文栈
ECStack = [
    globalContext
]

// 全局执行上下文
global = {
    VO: [global],
    Scope: [globalContext.VO],
    this: globalContext.VO
}

// 函数foo被创建，保存作用域链到函数内部属性[[Scopes]]
foo.[[Scopes]] = [
    globalContext.VO
]
```

```js
// foo函数执行上下文
fooContext = {
    AO: {
        a: undefined,
        bar: function () {
            console.log(a)
        },
        arguments: []
    },
    Scope: [AO, globalContext.VO],
    this: undefined
}
```

```js
// bar 函数执行上下文
barContext = {
    AO: {
        a: undefined,
        arguments: []
    },
    Scope: [AO, globalContext.VO],
    this: undefined
}
```

当 `bar` 函数执行的时候，`foo` 函数上下文已经被销毁了啊（即从执行上下文栈中被弹出），怎么还会读取到 `foo` 作用域下的 `a` 值呢？

当我们了解了具体的执行过程后，我们知道 `bar` 函数执行上下文维护了一个作用域链：

```js
barContext = {
    Scope: [AO, fooContext.AO, globalContext.VO],
}
```

对的，就是因为这个作用域链，`bar` 函数依然可以读取到 `fooContext.AO` 的值，说明当 `bar` 函数引用了 `fooContext.AO` 中的值的时候，即使 `fooContext` 被销毁了，但是 JavaScript 依然会让 `fooContext.AO` 活在内存中，`bar` 函数依然可以通过 `bar` 函数的作用域链找到它，正是因为 JavaScript 做到了这一点，从而实现了闭包这个概念。

## 常见形式

常见闭包：

* 函数嵌套：函数里面的函数能够保证外面的函数的作用域不会被销毁，所以无论是在函数里面还是在外面调用函数里面的函数都可以访问到外层函数的作用域，具体做法可以将里面函数当做返回值返回后通过两次的括号调用
* 回调函数：回调函数会保留当前外层的作用域，然后回调到另一个地方执行，执行的时候就是闭包
* 匿名函数自执行：严格算也不是闭包，就是 `(function(){})()` 这种格式

## 优缺点

闭包的优点能够让希望一个变量长期驻扎在内存之中成为可能，避免全局变量的污染，以及允许私有成员的存在。

闭包的缺点就是常驻内存会增大内存使用量，并且使用不当容易造成内存泄漏。

如果不是因为某些特殊任务而需要闭包，在没有必要的情况下，在其他函数中创建函数是不明智的，因为闭包对脚本性能具有负面影响，包括处理速度和内存消耗。






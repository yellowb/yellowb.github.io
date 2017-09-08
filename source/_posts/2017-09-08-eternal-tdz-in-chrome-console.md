---
title: Chrome Console里的永恒死区/Eternal TDZ in Chrome Console
date: 2017-09-08 14:52:33
tags:
- JavaScript
- Chrome
---
# 问题
前段时间自己玩的时候, 发现如果在Chrome控制台依次输入以下代码，会出现比较奇怪的结果：
```javascript
let [a = b, b = 1] = [];    // 故意的, 会抛异常

typeof a;

let a = 'other value';

a = 'other value';
```

代码运行效果如下图：
![](/uploads/2017-09-08-eternal-tdz-in-chrome-console.png)

1. 首先第一行代码抛出`Uncaught ReferenceError: b is not defined`是比较容易理解的，因为第一行的**ES6 Destructing**语法等号右边是一个空数组，没有办法跟左边进行模式匹配，所以变量a和b会尝试用默认值赋值。然后代码`a = b, b = 1`以从左到右的顺序执行，由于执行到`a = b`时b还没有被**初始化(Initialised, 注意这里的用词)**，所以想用b给a赋值时发生了ReferenceError。

2. 然后第2行代码尝试获取a，抛出`Uncaught ReferenceError: a is not defined`，难道是因为第一句代码出错了，导致a没有定义？

3. 既然提示a没有定义，那么第3行就尝试定义a，但是却抛出`Uncaught SyntaxError: Identifier 'a' has already been declared`。刚刚第2行不是提示a没被定义吗？怎么这里就提示a已被声明了呢？

4. OK，既然又提示a已被声明，那我就去给它赋值呗...然而第4行又出错了，抛出的错误跟第2行一样。

程序运行到这里，最奇怪的地方就是：a这个变量是处于一种既没有defined，但是又被declared的状态，而且没有办法对它进行赋值，似乎a已经陷入一个死局，我们再也没办法使用它了。

# 分析
其实第一眼看到这些异常，自觉猜测可能跟let关键字的[暂时性死区](http://es6.ruanyifeng.com/#docs/let#暂时性死区 "暂时性死区")([TDZ](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let#Temporal_Dead_Zone_and_errors_with_let "TDZ"))有关，但并不确定Chrome的Console中的Scope到底是一个什么情况，于是把问题发上StackOverflow，[点击这里查看](https://stackoverflow.com/questions/46108143/strange-behavior-of-es6-destructuring-in-chrome-console "点击这里查看")。

按照大神[@Bergi](https://stackoverflow.com/users/1048572/bergi "@Bergi")的回答，结合他提供的另外2篇资料：[Chrome console already declared variables throw undefined reference errors for let](https://stackoverflow.com/questions/41255090/chrome-console-already-declared-variables-throw-undefined-reference-errors-for-l/41256149#41256149 "Chrome console already declared variables throw undefined reference errors for let") 和 [Are variables declared with let or const not hoisted in ES6?](https://stackoverflow.com/questions/31219420/are-variables-declared-with-let-or-const-not-hoisted-in-es6?noredirect=1&lq=1 "Are variables declared with let or const not hoisted in ES6?") ，大概理解并总结如下：

## 关于变量提升(hoisted)

首先，不管是`var`、`let`还是`const`，都会被JS引擎进行**变量提升(hoisted)**，也就是这个变量的**声明（declare）**会被提到当前Scope的最前面，而**初始化部分（initialization，一般是用等号进行赋值的语句）**则保持在原来的地方。注意这里讲的**声明**并不是形如`let b;`这样的代码，因为这句代码实质等于`let b = undefined;`，也就是它已经包含了**声明**与**初始化**两部分了。

而`var`、`let`和`const`在变量提升中的不同是：
1. 对`var`来说，如果在Scope开头 ~ 变量初始化部分之间尝试引用这个变量，并不会报错，**而是会返回undefined**.
2. 对`let`和`const`来说，做同样的事情则会抛出**ReferenceError**。

可以通过代码来帮助理解这个不同：
```javascript
function do_something() {
	/* bar 和 foo 的声明会被提升到此处 */
	console.log(bar); // undefined
	console.log(foo); // ReferenceError
	var bar = 1;
	let foo = 2;
}
```

## 关于暂时性死区（TDZ）
关于TDZ，其实就是对于用`let`和`const`定义的变量，在其Scope开头 ~ 变量初始化部分之间是不能访问的，一访问就会抛ReferenceError。这个不能访问的区域就叫TDZ。可通过下面的例子帮助理解：
```javascript
if (true) {
	// TDZ开始
	tmp = 'abc'; // ReferenceError
	console.log(tmp); // ReferenceError

	let tmp; // TDZ结束
	console.log(tmp); // undefined

	tmp = 123;
	console.log(tmp); // 123
}
```
更详细的资料可以看：[暂时性死区](http://es6.ruanyifeng.com/#docs/let#暂时性死区 "暂时性死区")([TDZ](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let#Temporal_Dead_Zone_and_errors_with_let "TDZ"))

## 如何理解一开始的问题？
回到一开始的问题，首先对于Chrome来说，直接在其Console定义的变量都将进入global scope。
1. 当执行`let [a = b, b = 1] = [];`时，JS引擎解析这句代码，我们意图定义两个变量a和b，所以就会在这里对a和b进行**声明**，则当前Scope里就存在a和b的声明了。
2. 然而第一句代码在执行时出错了，a和b没有被成功**初始化**，然而它们的**声明**还保留着。也就是说，a的TDZ开始了，但永远不会结束！
3. 接下来就很好理解了，对于第2和第4条代码来说，因为它们都意图访问a，然后a还没被**初始化**（还处在a的TDZ中），所以就报错`a is not defined`. 对于第3条代码来说，以为在这行代码之前a已经被**声明**了，所以不能重复声明。

OK，至此就能解释为什么Chrome会有这样的Output了。

另外大神还提供了一个例子可以reproduce这种情况：
```javascript
let b = (_ => { throw new Error() })();  // 抛异常
// b永恒的死区开始了...
```

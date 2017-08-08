---
title: 如何防止JavaScript属性被修改?
date: 2017-08-08 23:25:33
tags: JavaScript
---
# 背景
前段时间公司小伙伴问一个问题:

他在写前端JS时, 想防止用户对他挂到全局对象下的一个属性进行修改, 比如window对象下面挂了个属性"abc", 其值是一些比较重要的东西, 为了防止用户通过诸如浏览器控制台之类的途径对这个值进行修改, 有什么办法? (=_= 先不讨论他这种设计是否合理, 只探讨在此场景下的解决方案)

# 方法 - 常量属性
在对象上添加新属性的方法除了诸如`obj.abc = 123;`之类的赋值语句之外, 还可以用`Object.defineProperty`函数, 其输入参数为:
```javascript
Object.defineProperty(
	window,   //目标对象
	"abc",       //属性名
	{               //属性描述符
		value: 123, 
		writable: false, 
		configurable: false,
		enumerable: true
	}
); 
```
上面代码中最后一个参数为一个对象, 其中有一些配置项, 除了`value`之外, 其余都是控制访问行为的. 
1. 其中`writable`属性表示是否可以修改属性的值. 如果执行`window.abc =999;`,则这条语句在非严格模式下会静默失败, 严格模式下会抛出TypeError.
2. 其中`configurable`属性表示这个属性是否是可配置的. 也即是说是否允许**修改这个属性的属性描述符**(通过`Object.defineProperty`函数)或**删除这个属性**(通过`delete`)
3. `enumerable`属性表示属性是否可被枚举, 也就是是否会出现在`for...in...`中.

综上所述, 通过把`writable`属性和`configurable`属性设置为false, 就可以达到防止用户修改的目的.

# 额外的一些东西
Object也提供了另外一些有用的函数, 使得我们可以不仅仅把对象的属性变成常量, 而且可以把对象本身变成不可变.
## 1. Object.preventExtensions函数
`Object.preventExtensions`可以使得一个对象不能被添加新属性, 原有属性则会被保留.
```javascript
var obj = {a : 2};
Object.preventExtensions(obj);
obj.b = 3;
obj.b;   //undefined 
```
## 2. Object.freeze函数
`Object.freeze`会"冻结"一个对象, 除了`Object.preventExtensions`的效果外, 它还会把所有属性的`writable`属性和`configurable`属性设置为false. 这样一来就既不能修改对象原有属性也不能删除对象原有属性了. 它提供了最高级别的不可变性, 但要注意冻结的只是对象的属性, 至于这个属性所引用的另一个对象是不受影响的.
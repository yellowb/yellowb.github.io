---
title: 修复一个玩具性质WebIM系统OOM故障的过程与对IM系统设计的一些思考
date: 2018-08-10 16:52:33
tags:
- IM
- Bug
---

# 概述
分析并解决一个老旧的Web IM项目后台频繁OOM的问题，通过修改原先不合理的逻辑实现让Node.js峰值heap从超过2GB下降到100MB左右。

# 背景

公司内部有个2016年做的Web IM系统，技术栈如下：
1. 应用服务器：Node.js 4.2
2. 浏览器与应用服务器：Websocket
3. 数据库：Mongodb 3.2 + Mongoose 4.12

架构图大概如下面所示（实际上这个Web IM是一个大项目的一部分，不过这里只关注它就够了）：
![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20180810/system-arch.png)

可以看到Front-end server主要用于跟浏览器保持连接，Back-end server主要跟Mongodb打交道，这两个server之间用restful的方式通讯。

由于所属的那个大项目在2016年下半年就被砍了，所以这个Web IM也就停止开发了，但是由于还是有几个内部用户（目测不超过10人）在用，所以就一直半死不活地吊着。最近由于这个项目的Back-end server每天都会报OOM（out of memory），有时候一天报N次，故运维的人就找到我叫我看看啥问题。

实际上我根本没参与过这个Web IM的开发，接到这个任务时也一脸懵逼，以前开发这玩意的人也找不到了，只能硬着头皮上。

# 诊断
## Part1：OOM时发生了什么？
接到这个问题时第一个猜测是，一种可能是内存泄漏，一种可能是对象太多太大把堆撑爆了（不太希望是前一种）。直接从代码入手是不可能的，只能先看看Log了。直接搜索OOM的部分，能看到如下信息：

```
FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - process out of memory

<--- Last few GCs --->

79362148 ms: Scavenge 1396.6 (1458.1) -> 1396.6 (1458.1) MB, 1.1 / 0 ms (+ 2.4 ms in 1 steps since last GC) [allocation failure] [incremental marking delaying mark-sweep].
79363216 ms: Mark-sweep 1396.6 (1458.1) -> 1396.6 (1458.1) MB, 1068.1 / 0 ms (+ 3.5 ms in 2 steps since start of marking, biggest step 2.4 ms) [last resort gc].
79364283 ms: Mark-sweep 1396.6 (1458.1) -> 1396.6 (1458.1) MB, 1067.1 / 0 ms [last resort gc].


<--- JS stacktrace --->

==== JS stack trace =========================================

Security context: 0x34a3f5db4629 <JS Object>
    1: /* anonymous */(aka /* anonymous */) [/home/node/node_modules/mongoose/lib/document.js:1894] [pc=0x159cf1341fab] (this=0x34a3f5d041b9 <undefined>,pointCut=0x177d1bbf86f9 <String[4]: save>)
    2: arguments adaptor frame: 3->1
    3: InnerArrayForEach(aka InnerArrayForEach) [native array.js:~924] [pc=0x159d01f9d7df] (this=0x34a3f5d041b9 <undefined>,bk=0x147c5aaa64d9 <JS Function (SharedFu...
```

可以看到used heap已经去到14xxMB，印象中这个版本的Node.js默认Max Heap就是1.5GB左右，而且看GC已经释放不了空间了。继续看JS stack，可以看到是执行到

    /* anonymous */(aka /* anonymous */) [/home/node/node_modules/mongoose/lib/document.js:1894]
报错的，看起来像是跟Mongodb打交道时出的问题，继续看看document.js的1894行是什么：

```javascript
all.forEach(function(item) {
    if (!item) {
      return;
    }
    if (item.path.indexOf(lastPath) !== 0) {
      lastPath = item.path + '.';
      minimal.push(item);
      top = item;
    } else {
      // special case for top level MongooseArrays
      if (top.value && top.value._atomics && top.value.hasAtomics()) {
        // the `top` array itself and a sub path of `top` are being modified.
        // the only way to honor all of both modifications is through a $set
        // of entire array.
        top.value._atomics = {};
        top.value._atomics.$set = top.value;
      }
    }
  });
```

实际上是个遍历集合时用的匿名函数，这个代码块是包含在一个叫`$__dirty`的函数里，其的作用是为document对象的每个属性构造一个字符串形式的path（形如"a.b.c"），似乎是之后用于标记哪些field是dirty的，因为mongoose实际上会记录应用代码改过document对象的哪些field，之后update回mongodb时有用。

再稍微往上最终，可知这块逻辑是从Mongodb query数据后并向应用程序返回document array之前会调用的。**至此，可以大概推测OOM是在应用从Mongodb query数据时发生的，一个可能性是数据的size太大**。

## Part2：什么时候发生了OOM？
现在的疑问是，是一个怎样的query触发了OOM、哪里的代码触发了这个query、什么时候触发的？

搜索Log中OOM异常的上下文，可以发现每次OOM之前都有调用一个名为`getUnreadMsgs`的函数，并且从express的access log中看到这个函数居然返回了**几十MB**的数据，执行时间从**十几秒到五十几秒**不等！从字面上来看这个函数是去获取（某个用户？）未读的消息，看来要去看看数据模型长啥样的了。

由于是个半死不活的项目，公司也没有做太严格的限制，拿到账号密码后直接就连上Mongodb，其实Collection也只有几个，其中最重要的就2个：**msgs**（存储每一条消息的id、TopicId、发送时间、发送方、接收方、正文等）、**readMsgs**（记录接收者与已读消息的关系，包含消息id、接收者、读取时间等。如果某个接受者读了一条发给他的消息，那么就会往这里插入一条记录）。**readMsgs**数据量比**msgs**要大2个数量级。

这里有一点要提一下，这个Web IM似乎是根据Topic来组织聊天记录和参与者的，用户要聊天前必须先建立一个Topic，然后指定其他参与者，才可以开始聊天，并不支持单对单直接聊天，所以这里的Topic可以类比成其它一般IM里“群组”、“房间”这样的概念。

大概的数据模型关系如下：
![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20180810/data-model.png)

了解了数据模型，接下来就可以进`getUnreadMsgs`看看到底做了什么事情了。
首先这个函数的输入是一个userId，**第一步：**
```javascript
MsgModel.find({
        'receivers':{'$in':[userId]}
},
```
作用似乎是从**msgs collection**中把某个用户所有参与的聊天消息都拿出来。（看到这里的时候被震惊了一下，虽然说是个试验性的项目，但连基本的限流都没有... 试想聊天记录越来越多的情况）

**第二步：**
```javascript
ReadMsgModel.find({
        'userId':userId,
}
```
又被震惊了一下，不过也不难理解，上一步把用户所有相关的消息都取出来，那么为了过滤掉用户已经读过的，必然要从**readMsgs collection**中把该用户所有已读记录取出来了，然而这个collection的数据量比msgs要大得多...

**第三步：**
拿到用户所有**相关消息**和**已读记录**，用lodash里求差集的函数根据msgId把用户已读过的消息过滤掉，剩下的就是未读的了，返回给调用者即可。

总的来说逻辑是对的，但是不合理，做法十分粗放。**至此，可以大概推测OOM可能是因为getUnreadMsgs函数中从Mongodb中加载的数据量太大导致的**。

但是，是不是每次调用这个函数都会OOM呢？抱着疑问又去搜索了一下Log，发现并不是每次`getUnreadMsgs`执行时都会OOM，这个函数每日调用大概50次左右，但是OOM次数可能只有两三次。推测可能是每个用户的聊天消息数据量不一样，可能有些人比较多会触发OOM。好在Log中有记录每次调用时的userId，发现OOM之前的Log都会出现某一个特定用户，拿着这个userId去mongodb查，发现满足条件的msgs有**上万条**、readMsgs有**上百万条**！而其它用户基本都要少一半以上。

随便扫了一眼查询出来的readMsg collection中的数据，发现居然有多条数据**msgId和userId是相同的，但是readTime不一样的**！这里觉得非常奇怪，难道这系统还要记录用户每次是什么时候读一条消息的？

于是打开这个Web IM的UI，发现其并没有任何功能时需要用户以往每一次读消息的时间的，并且发现`getUnreadMsgs`函数是每次用户登录进系统时会执行一次，之后就没有其他触发点了，也就是说这个Web IM的设计**是用户登录时从数据库加载当前时间点之前的所有消息，之后的消息都是靠Websocket同步的（假设用户不关浏览器、不刷新、不重新登录）**。

至于为什么readMsgs collection中会存在**msgId是相同的，但是readTime不一样的**的数据，通过操作UI加上阅读代码，发现用户只要打开聊天界面中的某个**Topic**，系统就会Load出这个Topic下的所有消息并显示出来（包括已读和未读的），同时往readMsgs collection中新增改用户读过相关消息的记录，**不管这个用户之前是否已经读过**，也就是说，如果用户多次点同一个Topic，每一次系统都会为用户新增一堆已读记录，这就解析了先前那个问题。

另外看Log时还发现一个规律，就是`getUnreadMsgs`函数基本都是集中在早上6点多、中午11点多执行的，经过询问得知用户基本都是用这个系统跟澳洲的客户联系，而澳洲的时区比中国要早+1~+2个小时，也就是说中国早上6点多恰好是那边的8点多 - 上班时间。

通过阅读Log和代码、使用系统和跟用户沟通，**至此，可以大概推测OOM可能是因为某个用户的数据量特别大，从而导致`getUnreadMsgs`函数在用户登录时加载大量数据导致内存耗尽。同时知道了用户的使用习惯：早上和下午刚上班时会登录一次系统，之后都不会再重登录了**

## Part3：定位罪魁祸首
虽然我们推测是`getUnreadMsgs`函数引起OOM，但是还没有最终确认。而且这个函数里做了3步事情：
1. 从Mongodb中获取用户所有消息数据
2. 从Mongodb中获取用户所有已读消息的记录
3. 根据msgId，用第1步的数据减去第2步的数据得出用户未读消息。

我们并不知道是哪一步占用了最多的内存。这需要靠测试才能知道，于是搭建了一个本地的环境，通过增加以下2项收集内存占用信息：

1.通过在一些关键路径上打印当前heap占用情况：
```javascript
```javascript
winston.info('Memory usage for getUnreadMsg - 2', JSON.stringify(process.memoryUsage()));

// output: info: Memory usage {"rss":71290880,"heapTotal":50755680,"heapUsed":45254776}
```
```

2.为防止OOM影响测试，通过添加node启动参数把Max Heap调大到8GB。
```javascript
node --max_old_space_size=8192 server.js
```

通过用该用户id登录进行测试，发现第2步 - 从**readMsgs collection**中查找该用户所有已读记录，占用的内存最多，used heap增长超过4GB，其余两步对heap的增长只有几十MB。

**至此，可以推断`getUnreadMsgs`函数中第2步获取用户所有已读记录时占用了太多内存**

# 开刀
虽然我们大概定位了发生问题的代码，但是到底是因为数据量太大，还是由于mongoose有bug导致的呢？由于这里用的是Mongoose的一个稳定Release版本4.12，按理说不应该有这种问题，Google了下好像也没有人提相关的bug，如果要深入研究Mongoose的源码，修复成本就会比较高。所以这里选择了去针对数据量太大的方向做修正。

## Stage1：是否能清除老旧数据
因为这个Web IM主要是公司内部用户（客服）与少量外部用户（客户）交流的渠道，客户主要用途是询价、下单，数据量上大概每天新增几百条消息，一个十分少的量。按照常理推断，一般一个运单周期也就两三个星期，所以把两年前（2016）到现在的数据都加载出来其实是没意义的，用户也不关注那么久之前的东西。

直接跟用户沟通了一下，约定只取半年内的数据。如果加了时间限制，那么对于那个数据最多的userId，readMsg记录可以从**上百万条**减少到44万左右：

```javascript
// Mongoose query时加上这个条件，index也相应update
'readDateTime': {$gt: timeUtil.monthsPriorToCurrent(MONTH_LIMIT_FOR_CHAT_HISTORY)}

// timeUtil.monthsPriorToCurrent(MONTH_LIMIT_FOR_CHAT_HISTORY)是一个工具函数，返回是当前时间减去半年得到的Date对象
```

## Stage2：不取一些没有用的数据
从原先的代码可以发现，query mongodb时是没有加projection的，也就是把所有field都取出来，实际上从代码逻辑上看，有一些field是没有用的，所以应该在投影上把它们去掉以节省内存：

```javascript
ReadMsgModel.find({
        'userId':userId,
        'readDateTime': {$gt: timeUtil.monthsPriorToCurrent(MONTH_LIMIT_FOR_CHAT_HISTORY)}  // Reduce data vol
 },
        // /** 只取msgId足矣 **/
{
        '_id': 0,
        'msgId': 1
 }
```

## Stage3：适当调大Max Heap
由于线上环境是一台物理机跑N个docker容器的，所以这个值不能无限调大，暂定为默认的2倍，即3GB。

经过了上面几个优化，原先heap的**增幅从4GB下降到1.6GB左右**，配合调大Max Heap，部署后稳定运行了2日也没出现OOM了，从Log中看到最大Used Heap大约在1.9GB左右。但个人感觉还是不爽，就那么十几个人用的系统，居然还要那么多内存，这程序也太烂了吧，于是又想到了下面的方法。

## Stage4：用Mongo的distinct()代替find()
Mongodb自带有distinct命令，作用跟SQL的差不多。从程序逻辑上看，其实取出大量msgId相同的readMsg记录并没有什么卵用，因为后面做差集运算而已，这么多重复记录反而会成为负担。

另外Mongoose的find()返回的是Document对象数组，这个Document对象除了保护业务数据外，还有大量Mongoose自带的元数据（后面实测一个只含一个属性，其值是长度为24的字符串的Document对象大约占4KB内存），这实际上是非常大的浪费，因为有效载荷占总体积实在太少了。而distinct()函数不返回Document对象数组，而是直接返回字符串数组，应当会节省不少空间。

把原先代码：
```javascript
ReadMsgModel.find({
        'userId':userId,
        'readDateTime': {$gt: timeUtil.monthsPriorToCurrent(MONTH_LIMIT_FOR_CHAT_HISTORY)}  // Reduce data vol
 },
        // /** 只取msgId足矣 **/
{
        '_id': 0,
        'msgId': 1
 }
```

改为：

```javascript
ReadMsgModel.distinct('msgId', {
         'userId':userId,
         'readDateTime': {$gt: timeUtil.monthsPriorToCurrent(MONTH_LIMIT_FOR_CHAT_HISTORY)} 
}
```

这个改动效果十分明显，used heap涨幅直接**从1.6GB下降到30MB左右**！从返回数据上看，44万条ReadMsg只包含4k条不重复的msgId，重复率高达10000%....

部署后观察了两天，used heap基本不会超过100MB。这个API的响应时间也从几十秒变成不到1秒。

# 总结
通过以下实践，解决了OOM的问题：
1. 跟用户协商，放弃老旧数据
2. 使用投影，摒弃不用的属性
3. 使用distinct代替find

# 事后思考
最大感想，这东西TM就是个**玩具**啊... 做试验性项目也不能完全不顾performance吧，而且这项目还做了好几个月...

另外这个Web IM的数据建模也有非常大的问题，现代IM基本不会每个人 * 每条消息 * 每看一次都新增1条记录的吧，这数据量得直接n的3次方了。。。由于聊天消息天然就是有序的，并且发出之后就不能改变，实际上完全可以用一个游标记录某一个用户在某一个Topic里阅读到第几条消息即可，序号小于当前游标的为已读，大于的就是未读，即可大大节省存储空间，编程实现也很简单。。。

(https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20180810/new-data-model.png)

如果一定要用msgId集合运算，比如要实现用户能手动确认看了哪条消息，可能后面的看了，但前面的没确认，这种需求就无法用游标来实现了，得靠msgId做过滤。但是即使是这种情况完全可以用[Bloom filter](https://llimllib.github.io/bloomfilter-tutorial/ "Bloom filter")来实现啊，超少内存占用，超快运算速度有木有！

人还是要多学习！

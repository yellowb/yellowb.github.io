---
title: 对“属性不确定的对象”如何进行数据建模的思考(Part 1)
date: 2018-01-10 16:52:33
tags:
- Mongodb
---

# 背景

部门里有同学负责一个任务分派系统（以下简称**T系统**），其用户主要是公司里的客服人员。T系统的作用主要是为这些客服人员提供可视化的任务搜索、任务分派、任务追踪功能，简单来说用户打开T系统后就能看到自己手头上的任务或者新生成还没被人拿走的任务，然后可以对它们进行操作，并且在任务状态改变时可以在UI上看到他们的变化。

这里要着重点出的是：**T系统里有很多不同类型的任务，而不同类型的任务的属性是不一样的（会有少部分公共属性是一样的，其余大部分任务私有属性是不同的），并且用户还可以新增自定义任务和它们的属性。T系统还对用户提供了搜索功能能够基于任务属性做过滤。**数据模型例子：

```json
// 任务类型1
{
    // 公共属性
    type: "A",
    createTime: "20180101",
    // 私有属性
    p1: "v1",
    p2: "v2",
}

// 任务类型2
{
    // 公共属性
    type: "B",
    createTime: "20180102",
    // 私有属性
    p3: "v3",
    p4: "v4",
}
```
由于T系统是从别人手里接手过来的，拿到手时系统的基本框架、数据模型、数据流等都已经被确定了。基于公司高层对Oracle数据库的虔诚信仰，T系统底层的数据存储自然选用的就是Oracle RDBMS了（包括公司内部其它系统全都是Oracle RDBMS）。

首先，RDBMS能提供一个非常重要的性质：ACID，简单来说就是保证一个操作多个表的多行数据的事务是原子性的，也就是中间失败了前面的操作会回滚，并且提供了事务间的隔离性，防止读到脏数据。这个性质对维持数据的一致性非常重要，如果没有这个性质，那么应用程序就要自己去处理失败回滚的问题，这样会导致程序逻辑非常复杂。

回到T系统，它是怎么把任务数据存储到RDBMS里的呢？也就是它的数据建模是怎么做的呢？按照当前的设计，任务数据存放在2张表里：任务基本信息表（TaskBase表）、任务私有属性表（TaskDetail表）。
1. TaskBase表：存放任务PK、公共信息。每个任务在这张表里只有一条记录。
2. TaskDetail表：存放任务私有属性，每个任务的一个属性作为一个K-V对（也就是这个表有2列，一个列存放属性名，另一个列存放属性值）存放到这个表中，并通过FK关联到TaskBase表，指明属性是属于哪个任务的。一般一个任务都会有20+个属性。

具体例子可以参考下图：

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20180110/rdbms_ER.png)

为什么这样设计呢？个人估计是由于任务的私有属性种类在业务上是无法固化的，也就是可能随时变化的，每种任务的属性也不一样。对于这种“属性不确定的对象”，用RDBMS是无法很好的匹配的，主要有如下一些问题：
1. 因为RDBMS的结构是一张列基本确定的二维表，虽然可以对列做增删，但是数据量大时很麻烦。
2. 如果我们穷举所有任务的所有属性得到一个属性集，然后把属性集的每一个属性作为一个列，则我们会得到一个列数非常多的表，而且这张表可能非常稀疏（想象一下，如果对于一些属性，某一类型的任务是没有的，那么这些任务数据的相应列都是空的，进而造成这张表很多空值，浪费空间）。
3. 业务上这些任务的属性是可以被用户自定义的，如果用户每增加一个属性，我们就增加一个列，那么这个工作量是无法想象的。

所以当前T系统采用了一种折中的方案：把任务的私有属性放到另一张表中，用K-V的形式存储。然而这种设计也带来一些问题：
1. 单表数据量极大：由于每个任务可能有几十个属性，所以TaskDetail表会几十倍于TaskBase表，当前已经到达了好几亿的量级（这还是只保留最近几个月的数据的情况下），而且没有做任何的分表。按照以前看到阿里内部指南所说，不建议单表容量超过5M，当然我们用的是牛逼的Oracle数据库和高性能一体机，可能这个上限还能更高些，不过无论如何把上亿的数据塞到一个表中也不是什么好的实践。
2. 查询速度慢：查询有等值、范围、模糊几种，并且会在多个属性上做组合。虽然已经为TaskDetail表的Field和Value建立联合索引，但由于这个表数据量太大，即使走索引也要读取大量块。而且因为每个用户只会读他关注的那部分任务，所以看起来并不会有太热的数据，可能数据库缓存也不太好使。更别提会在一些Value上用Like来检索了...
3. 由于RDBMS每个列的数据类型都是预先定义的，为了兼顾到不同属性，这里Value列用的是varchar2，也就是说不管存储的是数值还是日期（实际上还是数值），都必须变成字符串，众所周知字符串消耗的空间要比数值大得多。（想象下"100000000" vs 100000000）

**总结一下：**
1. T系统把一些属性动态变化的对象变成K-V的形式存储到RDBMS。
2. 由于没有为检索做更多的优化，完全用DB来扛，导致性能堪忧。

# 一些思考
如果不对用户需求做任何改变，仅仅从数据存储和性能的层面来看，当前T系统的数据建模是否合适？

以前曾经参加过一次MongoDB的会议，会上有网易游戏的运维哥哥说到，以前他们用的是MySQL存储游戏角色数据，而游戏策划经常对角色属性进行更改（比如加技能、加装备等），每次变更都要对MySQL表加列，使得他们非常痛苦。后来他们迁移到MongoDB，由于MongoDB是Schemaless的，所以可以随时加新的属性，并且每条记录的属性都可以不一样，这个迁移让他们运维的难度大大降低，引用他们的原话就是“舒服多了”。

那么如果我们像网易游戏哥哥一样采用Schemaless的数据库来建模，是不是会有一些不同？下面先来对比一下不同类型数据库的特点。

## 不同类型数据库的对比
![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20180110/db-roadmap.jpg)

### 1.RDBMS
从上世纪70年代E.F.Codd的论文《A Relational Modelof Data for Large Shared Data Banks》和SQL语言被提出开始，到80年代各种商用数据库比如Oracle、MS SQL等的推出，RDBMS在银行、电信等行业得到广泛应用。

这种数据库的特点就是完全实现了上面论文的关系模型、支持SQL，并且提供ACID特性。对于应用来说，不需要关注事务失败回滚等细节，非常易于使用。然而这个时代的数据库基本是单节点的，要提升性能基本是靠Scale-Up的方式，也就是买更贵的硬件。

2000年之后，由于互联网的兴起，大型互联网公司需要存储的数据的量非常大，这时候如果还是靠买更贵的硬件来升级，成本上行不通，另外即使成本不是问题，最贵的硬件的性能也不能满足这些公司的需求了。这时候一些互联网公司纷纷投向MySQL，并通过中间件的方式，实现了把大量数据拆分到多个MySQL实例中，满足了对海量数据的存储要求。不过对于单个数据库节点来说，其自身仍然是一个单机版的数据库，只不过是靠中间件实现了路由把多个单节点数据库组合起来了。

如果套用分布式领域的[CAP理论](http://www.hollischuang.com/archives/666 "CAP理论")来分析，RDBMS支持的就是C（一致性）和A（可用性）。

### 2.NoSQL
互联网兴起后，特别是社交等应用的火爆，对存储系统的容量和吞吐量提出了更高的要求，这时候传统RDBMS已经不太适应。结合这类应用的特点（比如不需要强一致性、丢失一些数据是可以容忍的），NoSQL被提出来了。

NoSQL是一个模糊的概念，我的理解是为了满足一些特殊需求，在原先RDBMS支持的特性上做减法，去掉一些不需要的特性，从而产生的为某种目的特殊进化的数据库。比如为了拥有高水平扩展性，有些NoSQL数据库把事务隔离、Join、FK等特性舍弃；又比如为了追求高吞吐量，只支持K-V类型的数据结构并且只用内存存储。

NoSQL在互联网公司里被大量地使用，但是核心的（比如交易、订单等牵涉到金钱的，不允许出现数据不一致）业务也还是RDBMS居多，因为那些被NoSQL抛弃的原先RDBMS拥有的特性，实际上是转嫁到了应用层面去handle，而这个是非常复杂的。

### 3.NewSQL
基于上面的理由，这时候有人跳出来说要把SQL给找回来了，于是提出了NewSQL的概念。对于这个概念，个人理解是同时具有RDBMS的ACID特性 + 高水平扩展性的数据库，有些产品还包含内存引擎等等其它东西。对于这个东西，比较大的应用是Google的F1和阿里的Oceanbase（Oceanbase与NoSQL的区别见[这里](https://www.zhihu.com/question/19841579 "这里")）。然而这些系统当前都没开源，另外有一些开源系统比如HANA、国人的TiDB等，似乎还没有大型成功的产品案例。

## 使用Schemaless方式对T系统进行建模的尝试
这里选用的是一个典型的Schemaless数据库MongoDB（v3.4.5），GUI查询分析器用的是[MongoBooster](https://nosqlbooster.com/ "MongoBooster")，测试用的脚本代码在[这里](https://github.com/yellowb/mongodb_study/tree/master/modeling/chaosAttrs "这里")可以找到。

### 建模方式1：直接把任务的所有属性作为数据库记录的所有属性
这种是最简单最直接的建模方式，其实就是把业务上每个任务的每个属性直接映射到每条记录中去，因为MongoDB不需要预定义Schema，所以不同类型的任务可以直接存储在同一个Collection中。具体例子如下：
```json
// 任务1
{
    // 公共属性
    _id: 1,
    type: "A",
    createTime: "20180101",
    // 私有属性
    p1: "v1",
    p2: "v2",
}

// 任务2
{
    // 公共属性
    _id: 2,
    type: "B",
    createTime: "20180102",
    // 私有属性
    p3: "v3",
    p4: "v4",
}
```
可以看到每条记录都是一个扁平的结构，所有任务都存储在一个Collection中而不需要关系它们属性不同的问题，优点是结构简单明了容易理解，但是如果一旦牵涉到检索，缺点会非常明显，下面会详细阐述。

首先为了测试，写了一个准备测试数据的脚本[modeling.js](https://github.com/yellowb/mongodb_study/blob/master/modeling/chaosAttrs/modeling.js "modeling.js")，在这个脚本中，我预先定义了20种不同的field name，并提供了产生随机对象的函数，这个函数会产生一个包含6个属性的对象，这6个属性是从预先定义的20个field name中随机抽取的，并且它们的field value也是随机产生的字符串或数值。最终我会往MongoDB中插入100W个随机对象，用来模拟T系统中的任务。下面是一些随机对象的例子：
```json
/* 1 createdAt:1/10/2018, 3:11:05 PM*/
{
    "_id" : ObjectId("5a55bc89fa2fd435b01ab39f"),
    "weibo" : "GYEUY-F7PM8",
    "zhihu" : 8413.0,
    "length" : "2EVPQ-SYZ66",
    "fax" : "E85MP-2U71D",
    "facebook" : 9584.0
},

/* 2 createdAt:1/10/2018, 3:11:05 PM*/
{
    "_id" : ObjectId("5a55bc89fa2fd435b01ab3a0"),
    "grade" : "4G0GO-QXUT1",
    "zhihu" : 4534.0,
    "fax" : "6HV19-3NVZX",
    "weight" : 9027.0,
    "size" : "APER8-LGJG9",
    "height" : 4765.0
},
```
下面来测试下检索性能，我们通过分析执行计划来进行。代码是[modeling_query.js](https://github.com/yellowb/mongodb_study/blob/master/modeling/chaosAttrs/modeling_query.js "modeling_query.js")

首先在没有任何索引的情况下，我们尝试分析一个等值查询（weibo='GYEUY-F7PM8'）：
```javascript
db.modeling.find({weibo: 'GYEUY-F7PM8'}).explain('executionStats');
```
下面是执行计划：
```json
    "executionStats" : {
        "executionSuccess" : true,
        "nReturned" : 1,
        "executionTimeMillis" : 451,
        "totalKeysExamined" : 0,
        "totalDocsExamined" : 1000000,
        "executionStages" : {
            "stage" : "COLLSCAN",
            "filter" : {
                "weibo" : {
                    "$eq" : "GYEUY-F7PM8"
                }
            }
        }
    }
```
可以看到在一个没有索引的属性上进行等值查询，MongoDB的策略是COLLSCAN，也就是全表扫描，总共扫描了100W行记录，返回了1条记录，花费451ms，这个时间基本上已经很慢了，如果数据量再上一个数量级，那么时间更无法忍受。

解决办法看起来很简单，就是给weibo这个field建立索引就好了。建立完之后我们再跑一次查询：
```json
    "executionStats" : {
        "executionSuccess" : true,
        "nReturned" : 1,
        "executionTimeMillis" : 1,
        "totalKeysExamined" : 1,
        "totalDocsExamined" : 1,
        "executionStages" : {
            "stage" : "FETCH",
            "inputStage" : {
                "stage" : "IXSCAN",
                "nReturned" : 1,
                "keyPattern" : {
                    "weibo" : 1
                },
                "indexName" : "weibo_1",
                "indexBounds" : {
                    "weibo" : [
                        "[\"GYEUY-F7PM8\", \"GYEUY-F7PM8\"]"
                    ]
                }
            }
        }
    }
```
可以看到MongoDB的策略是IXSCAN，也就是通过索引检索，只扫描了1个索引项和1条记录就得到了结果，时间开销是1ms。范围查询的效果也是类似的，可以自己跑脚本里的代码。

可喜可贺！但是如果这时候用户说他又要在另一个Field上做检索，怎么办？我们可以给他在另一个Field上也建立索引。那如果用户说所有Field都要用来检索呢，每个Field建一次索引？先不提索引太多会导致插入变慢的问题，如果用户每新增一个Field，都要我们去人手维护索引，那么工作量也是相当可观。

### 建模方式2：把任务的每个属性作为K-V对象存储到属性数组中
在这种建模方式下，业务上每个任务还是对应到MongoDB中的一条记录，但是跟建模方式1不同的地方在于，我们不直接把任务的属性作为记录的顶层属性存储，而是在记录中创建一个Array，命名其为“attrs”（即Attributes），然后任务的每一个属性（Field name、value）作为一个对象存储到这个数组中。例子如下（代码在[modeling_2.js](https://github.com/yellowb/mongodb_study/blob/master/modeling/chaosAttrs/modeling_2.js "modeling_2.js")）：
```json
/* 1 createdAt:1/10/2018, 3:54:44 PM*/
{
    "_id" : ObjectId("5a55c6c482acdb08ecec64ba"),
    "attrs" : [
        {
            "facebook" : "LHXQC-PKVAP"
        },
        {
            "city" : 8961.0
        },
        {
            "email" : "HQUQM-UK04L"
        },
        {
            "facebook" : 5608.0
        },
        {
            "age" : "917PN-PFE82"
        },
        {
            "city" : 9439.0
        }
    ]
},
```
可以看到属性都被放进Array了。那么这样做有什么好处呢？首先要提到的是MongoDB可以对Array建立索引，实际上产生的效果就是对Array里每个元素建立了索引。而且，如果元素是一个对象，那么检索时，匹配条件就是看检索条件跟对象是不是"**完全**"相等，其中包括：Field Name、Field Value、Field的个数、Field的顺序。比如：
```javascript
// 相等
obj1 = {age: 18, name: 'A'}
obj2 = {age: 18, name: 'A'}

// 不相等
obj1 = {age: 18, name: 'A'}
obj2 = {name: 'A', age: 18}
obj3 = {name: 'A', age: 18, weight: 100}
```
基于这个性质，我们在"attrs"上建立索引，那么我们的等值查询可以这样写（代码在[modeling_2_query.js](https://github.com/yellowb/mongodb_study/blob/master/modeling/chaosAttrs/modeling_2_query.js "modeling_2_query.js")）：
```javascript
db.modeling_2.find({attrs: {"facebook" : "LHXQC-PKVAP"}}).explain('executionStats');
```
可以看到我们直接把检索条件（"facebook"="LHXQC-PKVAP"）作为一个对象传入了，那么MongoDB就会通过索引为我们完全匹配这个对象，找出包含attrs Array中包含{"facebook" : "LHXQC-PKVAP"}这个对象的所有记录。执行计划如下：
```json
    "executionStats" : {
        "executionSuccess" : true,
        "nReturned" : 1,
        "executionTimeMillis" : 0,
        "totalKeysExamined" : 1,
        "totalDocsExamined" : 1,
        "executionStages" : {
            "inputStage" : {
                "stage" : "IXSCAN",
                "nReturned" : 1,
                "keyPattern" : {
                    "attrs" : 1
                },
                "indexName" : "attrs_1",
                "indexBounds" : {
                    "attrs" : [
                        "[{ facebook: \"LHXQC-PKVAP\" }, { facebook: \"LHXQC-PKVAP\" }]"
                    ]
                }
            }
        }
    }
```
可以看到只扫描了一个索引项，花费1ms就返回了。如果不加索引，这里要差不多1秒。

似乎我们解决了快速检索动态变化的属性的问题，但是我们还没测试范围检索呢。范围检索的写法如下，比如我们想找city>9500的记录：
```javascript
db.modeling_2.find({attrs: {$elemMatch: {"city": {$gt: 9500}}}});
```
但是会发现，这时候索引是不生效的，会走全表扫描。主要原因是MongoDB是把整个属性对象作为一个整体来索引的，如果查询要对比这个对象里的一部分，这时候就无法走索引了，如果一定要支持这种范围检索，就得单独为city这种属性建立索引：
```javascript
db.modeling_2.createIndex({'attrs.city': 1});
```
这样子对于city>9500这种查询才会生效。

# 总结
上面我们探讨了一下用Schemaless数据库对T系统进行建模的思路，对于等值查询来说，是有一种比较完美的方案可以做的，但是对于范围搜索，似乎还没有一种简洁的方案。
另外这里只是对建模进行思考，实际项目还要考虑数据一致性、事务等问题，这些东西MongoDB只支持到单条记录级别的，所以完全用它来代替肯定会有不少问题，还是期待NewSQL的实用化吧：）

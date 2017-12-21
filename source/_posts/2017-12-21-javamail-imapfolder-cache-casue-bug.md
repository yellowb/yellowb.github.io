---
title: 史前巨坑之Javamail IMAPFolder类中的Cache导致打开的Folder Session某些情况下不能获取邮件变化
date: 2017-12-21 15:52:33
tags:
- Java
- Javamail
- Bugfix
---
# 背景
厂里有一个Java项目，需要走IMAP协议去邮箱读取邮件，于是自然使用了Javamail这个库。然后为了减少频繁打开/关闭Session连接的性能消耗、限制多线程环境下每个邮箱文件夹被打开的Session数量，就基于Apache Commons-pool自己封装了一个Session连接池。

简单来说连接池里的每个连接都是一个活动的Session（其实就是`com.sun.mail.imap.IMAPFolder`类的对象，因为在Javamail中一个Session只能对应一个文件夹），每次代码需要操作邮件时，从连接池中取出一个IMAPFolder对象，用完后归还回连接池里。

前段时间刚上线邮件量不太大时，似乎没什么大问题（现在想想其实应该一直都有问题，只是邮件量不大时，被后台一个定时做Recovery的Job掩盖了）。最近邮件量大了后开始频繁出现一个奇怪的问题：**有时候扫描邮箱想取最新的邮件时，发现取不到，但实际上这时新邮件已经到达邮箱了。也就是邮箱里的邮件和Java代码这边看到的情况不同步**，而且这个问题并不是总是出现的，往往重启程序后一开始是没问题的，但运行了一段时间就会不定时地出现。

# 分析
一开始以为是项目里代码写的有问题，根据经验这种一开始运行没问题但运行了一段时间后开始不定时出现奇怪问题的现象，大多由于某些数据被错误地覆写（临界区、内存泄露等）导致。所以一开始以为又是多线程的问题，但是盯着代码看半天也没发现哪里不对，由于多线程问题本地又比较难模拟场景，Production打印的Log又很少无法提供分析依据，于是Bug Fix过程一度陷入僵局。

后来得到thoughtworks的同学提醒，`IMAPFolder`类中有这样一个成员变量（以下代码都基于Javamail v1.5.6）：

```java
protected MessageCache messageCache;// message cache
```
`messageCache`是什么鬼？点进去继续看：
```java
public class MessageCache {
    /*
     * The array of IMAPMessage objects.  Elements of the array might
     * be null if no one has asked for the message.  The array expands
     * as needed and might be larger than the number of messages in the
     * folder.  The "size" field indicates the number of entries that
     * are valid.
     */
    private IMAPMessage[] messages;
```
WTF？！难道还真缓存了邮箱的Message？为了一探究竟，眼睛看代码不是好办法，用IDEA把源码下下来attach上去就开始Debug。

(以下测试代码可以从[这里](https://github.com/yellowb/email-engine-poc/blob/master/PocRedis/src/main/java/test/email/TestSearchMailByMsgID.java "这里")找到)

(1) 第一个问题，这个`messageCache`是什么时候初始化的？
根据Debug，发现`IMAPFolder`有个open函数：
```java
public synchronized List<MailEvent> open(int mode, ResyncData rd)
				throws MessagingException {
	// 一堆检查...
	
	    // Initialize stuff.
	    opened = true;
	    reallyClosed = false;
	    this.mode = mi.mode;
	    availableFlags = mi.availableFlags;
	    permanentFlags = mi.permanentFlags;
	    total = realTotal = mi.total;
	    recent = mi.recent;
	    uidvalidity = mi.uidvalidity;
	    uidnext = mi.uidnext;
	    highestmodseq = mi.highestmodseq;

	    // Create the message cache of appropriate size
	    messageCache = new MessageCache(this, (IMAPStore)store, total);
```
看来是open一个Folder时去初始化的，构造函数的this即Folder本身，total是之前代码获取的邮箱中邮件的总数。再看看这个构造函数干了啥：
```java
    /**
     * Construct a new message cache of the indicated size.
     */
    MessageCache(IMAPFolder folder, IMAPStore store, int size) {
	this.folder = folder;
	logger = folder.logger.getSubLogger("messagecache", "DEBUG IMAP MC",
						store.getMessageCacheDebug());
	if (logger.isLoggable(Level.CONFIG))
	    logger.config("create cache of size " + size);
	ensureCapacity(size, 1);
    }
```
实际上有用的只有最后一行，调用了ensureCapacity函数：
```java
/*
     * Make sure the arrays are at least big enough to hold
     * "newsize" messages.
     */
    private void ensureCapacity(int newsize, int newSeqNum) {
	if (messages == null)
	    messages = new IMAPMessage[newsize + SLOP];
	else if { ...
```
这个message是messageCache的成员变量，类型是一个IMAPMessage数组，其实它就是实际上缓存Message对象的地方。由于messageCache在初始化时，message肯定是null，所以会走到第一个if分支，最后就是new了一个空数组而已（准确来说是数组里每个元素都是null），数组长度是刚刚获取到的当前邮件数量 + SLOP。（SLOP的值是64，其实就是预先提前多分配64个位置，避免以后每次扩容时都要new一个新数组）。**这里就会有2个疑问了：第一是为什么只new了一个空数组而不填充里面的值？第二是这个缓存居然会扩容，那么它什么情况下会扩容？**

这里先解决第一个问题：为什么只new了一个空数组而不填充里面的值？
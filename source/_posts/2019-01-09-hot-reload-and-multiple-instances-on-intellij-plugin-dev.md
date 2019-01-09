---
title: IntelliJ插件开发环境搭建:代码热部署与多开实例
date: 2019-01-09 16:52:33
tags:
- IntelliJ
- Plugin
---

# 摘要
默认情况下的IntelliJ环境插件开发方式不支持在一个IDE中启动多个IntelliJ实例和代码热部署。本文着重解决以下2个问题：
1. 如何便捷地在一个IDE中启动多个插件（IntelliJ）实例；
2. 如何配置JRebel使得代码被修改后能热部署到运行的实例中，不须每次改完代码都重启实例。这里要求IntelliJ的版本不得比**2018.1**新，否则JRebel无法启动，原因后面会分析。

# 背景
## 项目需求
想做一个IntelliJ插件，这个插件需要具有**点到点通讯**的能力（含单对单、单对多，也就是A、B两台机器的IntelliJ安装了这个插件后，这两个分别位于A和B上的插件实例能通过网络交换数据）。

## 一些IntelliJ插件开发的基本概念
IntelliJ插件的开发需要在IntelliJ中创建一种名为**“IntelliJ Platform Plugin”**的工程，这里不讲如何开发，如要学习请移步至[官网](http://www.jetbrains.org/intellij/sdk/docs/basics.html "官网")，以下内容默认你懂得最基本的插件开发知识。这里要提的是在开发过程中，每次运行与调试都会启动另外一个IntelliJ实例（或者叫Sandbox），你写的插件代码会被打包到这个新启动的IntelliJ实例里运行，前后两个IntelliJ实例不会相互影响，它们的关系如下图：

`[写插件代码的IntelliJ] ---Run/Debug插件---> [运行插件代码的IntelliJ (Sandbox)] `

这种形式有点像做Web开发时，把代码打包到Web容器中运行的模式，只不过在这里容器是另一个IntelliJ实例而已。

## 待解决的痛点
1. 由于IntelliJ默认情况下无法支持**代码热部署**，导致修改完代码之后想看效果必须重新启动IntelliJ实例，但是IntelliJ的启动时间相对来说还是比较长的，这样每次都要浪费很多等待时间会让开发过程比较痛苦，如果能够做到修改代码不用重启IntelliJ实例就能看到效果，那么开发效率会大大提高。
2. 基于功能需求，必须要在我的本地开发环境上能够启动2个或以上的IntelliJ实例，来模拟网络交互，但是IntelliJ默认只能启动一个，所以必须找到办法来支持**“多开”**。

# 痛点一: 代码热部署
这里使用了JRebel来实现开发时的代码热部署。JRebel是[ZeroTurnaround公司](https://zeroturnaround.com/ "ZeroTurnaround公司")的产品，主要功能是通过实现JVM上class的热替换来缩短开发时等待容器重启动的时间，支持各种Java Web容器，Tomcat、Weblogic等... 然而它的license非常贵，1人 * 1年的价格高达550美金，激活前可以免费用30日，不过公司买了它的license，不用白不用 :)

JRebel的配置还是比较简单的，我参考了这个[blog](https://benkiew.wordpress.com/2017/09/09/how-to-develop-your-intellij-idea-plugins-even-faster-with-jrebel/ "blog")。但是却遇到JRebel无法运行的问题，所以在给出具体配置步骤前，先来看看这个问题。

## 问题
由于之前我一直打开IntelliJ IDEA的Auto Update功能，所以用的都是最新版本（本文写作时间为2019-01-09，最新版本为2018.3.1[Build: 183.4588.61]）。当我把JRebel激活、配置到项目的Run Configuration后尝试运行插件，却报下面这个错误：

```bash
2019-01-09 11:25:09 JRebel:  Starting logging to file: C:\Users\###\.jrebel\jrebel.log
2019-01-09 11:25:09 JRebel:  
2019-01-09 11:25:09 JRebel:  #############################################################
2019-01-09 11:25:09 JRebel:  
2019-01-09 11:25:09 JRebel:  JRebel Agent 2018.2.5-SNAPSHOT (201901081547)
2019-01-09 11:25:09 JRebel:  (c) Copyright ZeroTurnaround AS, Estonia, Tartu.
2019-01-09 11:25:09 JRebel:  
2019-01-09 11:25:09 JRebel:  Over the last 1 days JRebel prevented
2019-01-09 11:25:09 JRebel:  at least 0 redeploys/restarts saving you about 0 hours.
2019-01-09 11:25:09 JRebel:  
2019-01-09 11:25:09 JRebel:  License acquired from License Server: ###
2019-01-09 11:25:09 JRebel:  
2019-01-09 11:25:09 JRebel:  Licensed to ###.
2019-01-09 11:25:09 JRebel:  
2019-01-09 11:25:09 JRebel:  
2019-01-09 11:25:09 JRebel:  #############################################################
2019-01-09 11:25:09 JRebel:  
2019-01-09 11:25:10 JRebel: ERROR Class 'com.intellij.util.lang.UrlClassLoader' could not be processed by org.zeroturnaround.javarebel.integration.intellij.IntelliJClassLoaderClassBytecodeProcessor@null: org.zeroturnaround.bundled.javassist.CannotCompileException: [source error] getResource(java.lang.String,boolean) not found in com.intellij.util.lang.ClassPath
  at org.zeroturnaround.bundled.javassist.CtNewMethod.make(SourceFile:84)
  at org.zeroturnaround.bundled.javassist.CtNewMethod.make(SourceFile:50)
  at org.zeroturnaround.javarebel.integration.support.CBPs$DirectProcessorImpl.addMethod(SourceFile:210)
  at org.zeroturnaround.javarebel.integration.intellij.IntelliJClassLoaderClassBytecodeProcessor.process(SourceFile:28)
  at org.zeroturnaround.javarebel.integration.support.JavassistClassBytecodeProcessor.process(SourceFile:111)
  at org.zeroturnaround.javarebel.integration.support.CacheAwareJavassistClassBytecodeProcessor.process(SourceFile:34)
  at org.zeroturnaround.javarebel.integration.support.JavassistClassBytecodeProcessor.process(SourceFile:75)
  at com.zeroturnaround.javarebel.xy.a(SourceFile:372)
  at com.zeroturnaround.javarebel.xy.a(SourceFile:299)
  at com.zeroturnaround.javarebel.SDKIntegrationImpl.runBytecodeProcessors(SourceFile:47)
  at com.zeroturnaround.javarebel.vf.transform(SourceFile:134)
  at java.lang.ClassLoader.defineClass(ClassLoader.java:42009)
  at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
  at java.net.URLClassLoader.defineClass(URLClassLoader.java:467)
  at java.net.URLClassLoader.access$100(URLClassLoader.java:73)
  at java.net.URLClassLoader$1.run(URLClassLoader.java:368)
  at java.net.URLClassLoader$1.run(URLClassLoader.java:362)
  at java.security.AccessController.doPrivileged(Native Method)
  at java.net.URLClassLoader.findClass(URLClassLoader.java:361)
  at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
  at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:338)
  at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
  at com.intellij.ide.Bootstrap.main(Bootstrap.java:19)
  at com.intellij.idea.Main.main(Main.java:60)
Caused by: compile error: getResource(java.lang.String,boolean) not found in com.intellij.util.lang.ClassPath
  at org.zeroturnaround.bundled.javassist.compiler.TypeChecker.atMethodCallCore(SourceFile:777)
  at org.zeroturnaround.bundled.javassist.compiler.TypeChecker.atCallExpr(SourceFile:723)
  at org.zeroturnaround.bundled.javassist.compiler.JvstTypeChecker.atCallExpr(SourceFile:170)
  at org.zeroturnaround.bundled.javassist.compiler.ast.CallExpr.accept(SourceFile:49)
  at org.zeroturnaround.bundled.javassist.compiler.CodeGen.doTypeCheck(SourceFile:266)
  at org.zeroturnaround.bundled.javassist.compiler.CodeGen.atDeclarator(SourceFile:773)
  at org.zeroturnaround.bundled.javassist.compiler.ast.Declarator.accept(SourceFile:103)
  at org.zeroturnaround.bundled.javassist.compiler.CodeGen.atStmnt(SourceFile:383)
  at org.zeroturnaround.bundled.javassist.compiler.ast.Stmnt.accept(SourceFile:53)
  at org.zeroturnaround.bundled.javassist.compiler.CodeGen.atStmnt(SourceFile:383)
  at org.zeroturnaround.bundled.javassist.compiler.ast.Stmnt.accept(SourceFile:53)
  at org.zeroturnaround.bundled.javassist.compiler.CodeGen.atMethodBody(SourceFile:321)
  at org.zeroturnaround.bundled.javassist.compiler.CodeGen.atMethodDecl(SourceFile:303)
  at org.zeroturnaround.bundled.javassist.compiler.ast.MethodDecl.accept(SourceFile:47)
  at org.zeroturnaround.bundled.javassist.compiler.Javac.compileMethod(SourceFile:175)
  at org.zeroturnaround.bundled.javassist.compiler.Javac.compile(SourceFile:102)
  at org.zeroturnaround.bundled.javassist.CtNewMethod.make(SourceFile:79)
  ... 23 more
```

看到这个一脸懵逼，把错误信息敲入Google也找不到有用的信息。看了一下错误信息，说的是一个叫`com.intellij.util.lang.ClassPath`的类里有个`getResource(java.lang.String,boolean)`方法找不到。心想难道是JRebel依赖了IntelliJ自身的类，但是这些类的方法签名变了导致不兼容了？

于是先尝试了升级JRebel的版本，当前使用的已经是最新的stable版本（2018.2.4），尝试切换到更新的nightly build版本，还是同样的错误。

如果一方升级还是不兼容，那能不能让另一份（IntelliJ）降级呢？IntelliJ的官网上是可以找到旧版本的，但是应该用哪个版本呢？考虑到IntelliJ的社区版是开源的，于是上Github找到这个类：[ClassPath.java](https://github.com/JetBrains/intellij-community/blob/master/platform/util/src/com/intellij/util/lang/ClassPath.java "ClassPath.java")，通过它的历史，发现在2018年4月的某次[commit](https://github.com/JetBrains/intellij-community/commit/66101d0dc4f5821168699b058623981476322ace "commit")中，把原先存在的`getResource(java.lang.String,boolean)`方法改成`getResource(java.lang.String)`了：

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/github_1.png)

看这次commit的tags，可以看到都有哪些版本的IntelliJ包含了这个代码，其中最旧的build号为`idea/182.5107.16`。于是上IntelliJ的[下载页面](https://www.jetbrains.com/idea/download/previous.html "下载页面")，可以查到`idea/182.5107.16`对应的就是**2018.2.6**这个版本，也就是说我们只需要用**2018.1.7**就可以了。

## 正确配置步骤
（1）在IntelliJ的[下载页面](https://www.jetbrains.com/idea/download/previous.html "下载页面")下载**2018.1.7**版本并安装。收费版和社区版皆可，我这里下的是社区版，因为开发插件用不上收费版多出来的功能。

（2）在Settings中安装JRebel的插件，免费试用30日或正确激活：
![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/jrebel_setup_0.png)

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/jrebel_setup_1.png)

（3）由于我们需要用命令行参数的方式启动JRebel，所以需要找到JRebel插件安装后自带的核心动态链接库的路径（windows下使用jrebel64.dll，Linux下用so，MAC下用dylib）

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/jrebel_setup_2.png)

（4）在项目的Run Configuration中的VM启动参数最后加上
```bash
-agentpath:<你的路径>\jrebel64.dll
```

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/jrebel_setup_3.png)

（5）这样配置就完成了。可以测试一下，先启动插件，可以看到控制台有JRebel正常启动的信息：

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/jrebel_demo.png)

（6）这时候修改一下代码，把原先输出到对话框的字符串从“Hi”改成“Hello”，保存并编译一下，可以看到控制台会输出JRebel热替换的信息，并且对话框的字符串的确变了：

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/jrebel_demo_0.png)

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/jrebel_demo_1.png)

到此为止，热部署已经配置完成，每次代码修改再也不用痛苦地等IntelliJ重启啦！

# 痛点二: IntelliJ实例多开
之前已经提到过，默认IntelliJ中做插件开发时（同一个工程）只能启动一个Sandbox IntelliJ实例，但是我的需求需要两个以上的实例同时运行，一种比较笨的办法是把源码复制一遍作为另一个工程，但是这样两边代码同步会很麻烦。参考了一个外国人提出的[想法](https://intellij-support.jetbrains.com/hc/en-us/community/posts/206764005-Running-two-IDEA-instances-with-my-plugin "想法")，整理出一个可以方便地多开又不用维护多份代码的方法：

（1）在当前编写插件的IntelliJ中New一个空项目，作为“**镜像项目**”：

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/double_instances_0.png)

（2）在镜像项目中把原先的插件项目引入作为一个模块：

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/double_instances_1.png)

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/double_instances_2.png)

（3）配置好镜像项目的编译版本（Java 8）

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/double_instances_3.png)

（4）重点：新建一个新的SDK（必须跟之前用的区分开），并且sandbox home也要跟之前的SDK不一样。也就是说之后启动的两个Sandbox IntelliJ实例，都是在独立的Sandbox环境里

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/double_instances_4.png)

（5）在Project页面指定好镜像项目使用的SDK为刚刚新建的SDK：

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/double_instances_5.png)

（6）其它配置诸如Run Configuration、JRebel都是跟之前类似的，这里不再赘述。这样你就可以分别在原先项目的IntelliJ和镜像项目的IntelliJ中分别运行插件了！而且他们share同一份代码，你在任意一个地方改了，另一个地方也会更新到！

![](https://raw.githubusercontent.com/yellowb/yellowb.github.io/hexo/source/uploads/20190109/double_instances_9.png)



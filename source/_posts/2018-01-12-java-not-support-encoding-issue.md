---
title: 关于Java不支持cp932编码异常"java.nio.charset.UnsupportedCharsetException: cp932"的解决办法
date: 2018-01-12 15:00:00
tags:
- Java
---

# 背景

有一个通过Javamail去邮箱读取邮件的Java程序，运行的时候时不时会抛出这种异常：
```java
Exception in thread "main" java.nio.charset.UnsupportedCharsetException: cp932
    at java.nio.charset.Charset.forName(Charset.java:531)
```
查看日志，发生这种异常时都是读取到有日文的邮件。

# 分析
根据个人经验，应该是解码邮件正文时，邮件的编码格式设定成cp932，故程序尝试用这种字符编码去解码邮件正文，但是缺发现没有一个字符编码名为“cp932”。

那么cp932到底是什么东西呢？上网搜索了一下，[wiki](https://en.wikipedia.org/wiki/Code_page_932_(Microsoft_Windows)显示这种字符编码是用在日语环境的，这个也跟先前查看出错邮件发现是日文的情况吻合。那么为什么这种编码在Java中是不支持的呢？

我们可以查看JDK源码里的[这个类](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7-b147/sun/nio/cs/ext/ExtendedCharsets.java/#159 "这个类")，这个类是定义了所有扩展字符编码和它们的alias的。全局搜索发现的确没有定义“cp932”这个字符串，但是查找“932”时，找到了如下定义：
```java
this.charset("windows-31j", "MS932",
             new String[]{"MS932", "windows-932", "csWindows31J"});
```
心想这个MS932是不是跟cp932有一定关系呢？之后又通过搜索，发现[SO上一个问答](https://stackoverflow.com/questions/18245279/java-io-unsupportedencodingexception-cp932 "SO上一个问答")，里面有提到cp932不是一个标准的alias，所以没被写进Java源码中，其实本质上它跟MS932是一样的。

# 解决办法
其实有两种方法解决这个问题：
## 第一种办法
其实就是显式地捕捉UnsupportedCharsetException这种异常，然后判断里面携带的异常编码名称是不是“cp932”，如果是的话，就显式地用“MS932”去解码，如果是其它异常，就走其它异常处理流程。

```java
        try {
            // 假设这里的逻辑是原先的 part.getContent()
            // ...
            // ...
            // 碰到cp932! 抛异常!
            throw new UnsupportedCharsetException("cp932");
        } catch (UnsupportedCharsetException ue) {
            // 先判断是不是真的cp932.
            if(ue.getCharsetName().equalsIgnoreCase("cp932")) {
                System.out.println("发现cp932编码!");
                // 通过获取输入流, 强行指定MS932编码格式做解码
                InputStream is = new ByteArrayInputStream(barray);  // 获得一个输入流, 在下面用自定义的编码格式做解码
                String s2 = null;
                try {
                    s2 = IOUtils.toString(is, Charset.forName("MS932"));    // JDK支持MS932!
                    System.out.println(s2);
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    IOUtils.closeQuietly(is);
                }
            }
            else {
                // 可能是其余我们没遇到的奇怪的编码!
                throw ue;
            }
```
## 第二种办法
第一种办法需要显式地捕捉判断这种异常，会导致很多地方都要包try-catch块，写法十分不优雅。其实我们可以自己定义一个CharsetProvider类，专门对应这种cp932编码，然后在这个CharsetProvider类中重新路由到MS932编码就可以了：

```java
import java.nio.charset.*;
import java.nio.charset.spi.*;
import java.util.*;

public class CP932CharsetProvider extends CharsetProvider {
    private static final String badCharset = "cp932";
    private static final String goodCharset = "MS932";

    public Charset charsetForName(String charset) {
        if (charset.equalsIgnoreCase(badCharset))
            return Charset.forName(goodCharset);
        return null;
    }

    public Iterator<Charset> charsets() {
        return Collections.emptyIterator();
    }
}
```
然后还要在项目的META-INF文件夹下，新建一个services文件夹，在其之下建一个文本文件名为java.nio.charset.spi.CharsetProvider。在这个文件中，把我们刚刚写的CharsetProvider类的全名（包括package）写在里面，比如我的：
```java
test.charset.CP932CharsetProvider
```
这样在其它Java代码中直接调用“cp932”编码就不会出错啦，最终会路由到MS932编码来解码。

所有代码在：
1. [Java代码](https://github.com/yellowb/email-engine-poc/tree/master/PocRedis/src/main/java/test/charset "Java代码")
2.  [META-INF](https://github.com/yellowb/email-engine-poc/tree/master/PocRedis/src/main/resources/META-INF/services "META-INF")


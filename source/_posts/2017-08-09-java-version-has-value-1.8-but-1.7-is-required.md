---
title: Registry key Error Java version has value '1.8', but '1.7' is required
date: 2017-08-09 11:27:33
tags:
- Java
- JDK
---
# 问题背景
公司最近把Java开发环境升级到Java8, 安装完JDK之后发现在cmd下运行java命令会报如下错误：
> Error: Registry key 'Software\JavaSoft\Java Runtime Environment'\CurrentVersion'
> has value '1.8', but '1.7' is required. 
Error: could not find java.dll 
Error: Could not find Java SE Runtime Environment.

本地环境：
> OS：Win7_x64
JDK：JDK8u131_x64JDK：JDK8u131_x64

# 解决办法
1. 把环境变量**Path**中的这一项（若存在）删掉：`C:\ProgramData\Oracle\Java\javapath` 貌似这一项是公司预装的JRE自动升级所存放的地方。
2. 把**C:\Windows\System32**目录下的`java.exe`, `javaw.exe` 和 `javaws.exe` 删除。
3. 重新打开一个新的cmd, 验证结果。

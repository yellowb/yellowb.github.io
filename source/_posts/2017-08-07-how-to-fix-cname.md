---
title: How to fix CNAME file issue in hexo-deployer-git / 如何解决Hexo Git Deploy之后CNAME文件丢失的问题
date: 2017-08-07 10:25:33
tags: Hexo
---
# 问题
如果用Github Pages搭建博客并且关联到一个自定义域名(如namecheap之类的域名提供商)时，一般都需要在Github Pages仓库的根目录下建议一个**CNAME**文件，文件内容是自定义域名。

同时如果是用Hexo生成静态网页的话，一般会用**hexo-deployer-git**作为部署到Github的工具。但是**hexo-deployer-git**默认会用**force push**来提交，也就是说它会覆盖掉仓库里原有的数据，包括CNAME文件。

# 解决方案

根据这个[Issue](https://github.com/hexojs/hexo-deployer-git/issues/26 "Issue")，可以把CNAME文件放一份在本地的Hexo工作目录（就是你运行hexo generate / hexo deploy的目录）下的\source目录下。这样每次执行`hexo deploy`部署到Github时，都会把这个文件拷贝一份上去。问题解决。

# Ref
如何在namecheap上购买域名并关联到Github Pages可以参考下面这个：
[http://davidensinger.com/2013/03/setting-the-dns-for-github-pages-on-namecheap/](http://davidensinger.com/2013/03/setting-the-dns-for-github-pages-on-namecheap/ "http://davidensinger.com/2013/03/setting-the-dns-for-github-pages-on-namecheap/")


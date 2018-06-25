---
title: 第一次开发Chrome扩展踩过的坑
date: 2018-06-25 18:26:04
tags:
- Chrome extension
---

# 背景

前段时间由于个人一个痛点，并且也没试过开发Chrome扩展，于是决定动手开始整。下面是项目背景：

> 背景：此插件是由个人痛点，并结合珠海公交微信公众号启发而做的。
> 
痛点：微信公众号上能查询某个公交路线上的车子运行的实时数据，但如果我在**用电脑**，比如在加班，过一段时间后要去赶公交，为了节省等车的时间，我想得知车子是不是快要到我上车的站点，由于微信公众号没有**到站提醒**功能，我必须不断刷手机微信，十分麻烦。考虑到用电脑时一般都会开着浏览器，浏览器插件也能实现一些数据抓取和弹出通知的功能，并且安装也比较方便，所以以浏览器插件的方式实现。
> 
这个Chrome扩展能实现如下功能:
1. 同时订阅多条公交线路，每条线路可以订阅多个不同站点；
2. 程序会在公交车接近指定站点时，在屏幕右下角弹出通知（暂设定是还有<=3站距离时触发通知，当然你觉得太烦也可以关掉通知功能）；
3. 可以查看订阅线路上所有公交车的运行状况(自动刷新)。

**项目的源码和使用方法可以在[Github](https://github.com/yellowb/zhuhai-bus-notifier "Github")上找到。**

由于程序的逻辑比较简单，无非就是用户的设定存储在localStorage（chrome extension API）中，再启动一个定时任务定时通过ajax从某些webservice拿数据，然后进行计算，计算结果通过UI显示出来，技术栈就是html + js + css，所以不会对程序是如何实现的进行详细记录，但是虽说技术栈简单，然而Google在其之伤又多加了些限制，导致跟一般的网页开发有点不一样，这篇文章主要是记录一些第一次做Chrome扩展时遇到的坑。

# 踩坑
## No.1 console.log()到底打印到哪里去了？

实际上开始着手做项目前，看官网教程前我先上网找了几个Sample观摩了下，有一个Sample安装到Chrome之后会在浏览器右上角多一个图标，鼠标点击会弹出一个UI，UI中其实就是html页面。我试着添加一些js代码，比如给按钮加listener去用console.log()打印点东西，这时候突然感觉懵逼了：**console.log()打印的东西到底在哪里？貌似也不在当前tab的console里...**

先说答案：弹出的UI有自己独立的console，需要在弹出的UI区域上点鼠标右键，在菜单中选“检查”，这时会弹出一个新的Chrome devTools窗口，在这个devTools的console中能看到代码打印的信息。参考[Stack Overflow](https://stackoverflow.com/questions/14858909/cannot-get-chrome-popup-js-to-use-console-log "Stack Overflow")。

实际上要搞清楚这个问题，需要弄明白Chrome扩展的组成部分和它们之间的关系。一般可以包含以下几部分（只罗列本项目包含的，实际上还可以有其它东西）：
1. **popup**：鼠标左键点击**Chrome右上角扩展的图标后弹出的UI**，实际上包含了一个入口html和相关的js、css。**有自己独立的DOM和js运行环境**（也就是不能用js直接访问到其它部分的js变量）。popup**每弹出一次都会初始化一次**html和js代码，然后一般失去焦点时UI会消失，同时html和js都被销毁，已打开的devTools会自动关闭。devTools用上面的方法打开。代码有改动只要重新弹出popup即可生效，不需要reload扩展。
2. **background**：可以理解成插件后台运行的逻辑（js代码），没有UI，在这个项目中我把数据初始化和定时任务放在这里。**有自己独立的DOM和js运行环境**，devTools需要从Chrome扩展管理页面进去点击该扩展的background.html选项打开。一般来说只要浏览器没关闭，background中的代码都会一直运行，但Chrome自己有机制会杀掉闲置扩展的background代码，**可以通过一些配置强行指定Chrome保持其一直运行**（后面会提到）。代码有改动需要手工在Chrome扩展管理页面**点reload按钮才能生效**，如果想清除localStorage的数据还要卸载再重新安装。
3. **options**：扩展可以提供一个“设定”页面给用户填数据，这个页面可以通过鼠标右键点击Chrome右上角扩展的图标，在弹出菜单中点击“选项”进入，其包含了一个入口html和相关的js、css。**有自己独立的DOM和js运行环境**，devTools在这个页面按F12即可打开。代码有改动只要刷新页面即可。

可以看到，每一部分都有自己独立的js运行环境，所以如果要查看console的话，也要进入相应的devTools才行。至于不同部分如何通讯，参考[消息机制](https://developer.chrome.com/extensions/messaging "消息机制")。

## No.2 为什么点击button不能触发onclick事件中的代码？
同样是上一个问题中的Sample，我尝试在popup中的html代码中添加一个button，并绑定一个onclick事件，让它触发js文件里的一个函数，类似这样的代码：
```html
<body>
<p id="demo">=a</p>
<button type="button" onclick="count()">Count</button>
</body>
```
但是点击button时对应的函数缺没有触发，console中有这么一行红字出现：
```
Refused to execute inline script because it violates the following Content Security Policy directive: "script-src 'self' chrome-extension-resource:".
```
[Stack Overflow给出的答案](https://stackoverflow.com/questions/17601615/the-chrome-extension-popup-is-not-working-click-events-are-not-handled "Stack Overflow给出的答案")显示原因是这种方式违反了Google的Content Security Policy，这个CSP规定了不能执行inline script。

所以说如果想给页面元素绑定事件，就用如下这种通过js代码的方式：

```javascript
var a=0;
function count() {
    a++;
    document.getElementById('demo').textContent = a;
}
document.getElementById('do-count').onclick = count;
```

## No.3 加载音效会让popup弹出变得非常缓慢？
因为想做一个功能就是公交快到站时用Chrome的notification API在桌面右下角弹出一个通知框，并播放一个小音效。播放影响基本就是参照[这里](https://stackoverflow.com/questions/14917531/how-to-implement-a-notification-popup-with-sound-in-chrome-extension "这里")，把音频文件加载成一个Audio对象，再在它身上调用play()就行了。
```javascript
let yourSound = new Audio('yourSound.mp3');
yourSound.play();
```
我用的音频文件大小100KB左右，是网上找的，原先代码是每次弹出通知时会加载一次这个音频文件，后来想着能不能在启动时就加载了后面不要每次都重复加载，就把上面的代码放进common代码中，但是由于common代码也被popup引用了，同时popup每次弹出都会初始化一次代码，所以导致每次点击图标后都要等1秒左右才能看到popup的UI（说实话我也不明白为什么多加载了一个100KB的音频会慢这么多，不加载时秒开）。

于是就不把音频文件的加载放到common中了，回退到每次弹出消息时再加载。

实际上最后项目中把这个音频删掉了，因为实际使用时发现公交车非常频密，导致音效老是响，烦死了（滑稽）

## No.4 background中写定时任务用哪种方式？
由于这个项目需要定时从webservice抓取数据，所以就牵涉到实现定时任务的问题。官方提供并推荐了一种基于消息触发的方式实现定时任务：[Alarm](https://developer.chrome.com/extensions/alarms "Alarm")。

但是这种方式有个问题：在development mode（也就是在Chrome中打开开发者模式用文件夹路径的方式安装扩展）下可以指定定时任务的时间间隔到秒级别，但是在production mode（也就是打包压缩发布到chrome store上），这个间隔最小只能到1分钟，官方给出的理由是基于性能考虑。

然而1分钟对于我这个应用来说有时候是一个过长的时间，对于有时候交通十分顺畅（比如深夜）的时候，1分钟足够公交车跑相当远一段距离了。经过调整和试用，感觉把时间间隔定在20s是一个比较合理的值（这完全是经验主义）。

后来用Google搜索相关资料，Stack Overflow上有大神提到如果确定定时任务是要永远跑下去的话，完全可以用js原生的setInterval来代替Chrome的Alarm API。另外一点就是要防止Chrome把background杀掉，这个可以指定manifest.json里的一个值（persistent = true）即可，不过得心里十分清楚这个程序不会占太多资源：

```json
"background": {
    "scripts": [
      "js/background/background-entry.js"
    ],
    "persistent": true
  },
```

最后是采用了Stack Overflow上提供的这种方式，经过观察浏览器不间断运行了一周时间，这个扩展也没什么异常发生，资源占用也没什么问题。





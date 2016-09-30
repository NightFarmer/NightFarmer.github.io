---
title: 使用Chocolatey包管理器一键搭建windows开发环境
date: 2016-09-30 13:28:59
tags: [干货]
category: 其他
---

最近腾讯开放内测的微信小程序火了，而官方支持IDE只有windows版和Mac版的，稍微研究了一下这个IDE发现是node-webkit开发的，理论上应该是跨平台的，但不知为何这个IDE并没有支持Linux环境。喜欢折腾的我当然是要尝试一下的，奈何是使用Ubuntu作为主力开发环境，所以只能重做一个windows系统了。
话说回来，重装系统之后最大的问题就是开发环境需要配置，比如JDK、各种IDE、python、nodejs、tomcat、mysql、以及各种小工具等等，而把一个新系统配置到能正常进行开发工作基本上要耗费半天时间，时间就是生命，怎能这样无情的浪费。
Chocolatey是一个类似于linux中apt-get和yum这样的工具，通过他可以自动获取到需要软件的下载地址以及安装脚本已完成自动安装，而本篇则记录如何使用Chocolatey来通过脚本文件实现一键搭建开发环境，就像这样：
```
initWineEnv.bat
```
<!-- more -->

Chocolatey的安装很简单，打开一个cmd命令行窗口执行以下命令
```cmd
@powershell -NoProfile -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"
```

安装完成后就可以使用choco命令来安装大部分主流的应用了，如：
```cmd
choco install jdk8
```
这样chocolatey就会从服务器上寻找jdk8的最新下载地址和安装脚本自动从官网进行下载并完成安装以及环境变量的配置。

不只**JDK**，包括IDE的下载和安装也可以交给chocolatey，比如**AndroidStudio**
```cmd
choco install androidstudio
```
以及**AndroidSDK**
```cmd
choco install android-sdk
```
和**Genymotion**
```cmd
choco install genymotion
```

到此Android开发所需要的都已经搭建完毕，如果你还有其他需求也可以通过chocolatey来代替
如：
**python**
```cmd
choco install python2
```

**nodejs**
```cmd
choco install nodejs 
```

**IDEA**
```cmd
choco install intellijidea-ultimate
```

**WebStorm **
```cmd
choco install webstorm 
```

**notepad++**
```cmd
choco install notepadplusplus 
```

**LICEcap**
```cmd
choco install licecap
```

**cmder**
```cmd
choco install cmder
```

**Sublime Text**
```cmd
choco install sublimetext2
```

**TortoiseSVN**
```cmd
choco install tortoisesvn
```

**git**
```cmd
choco install git
```

...

etc.

基本上所有windows上主流的工具或应用都可以使用chocolatey来进行安装。
可以通过https://chocolatey.org/packages 在官方网站中搜索，里面收录了4174个应用并且每时每刻都在增加着。

而我们在拿到一个新的windows操作系统后，只需要将我们想要安装的软件通过choco install命令放在一个bat文件中，双击bat文件或者使用命令行执行bat文件然后就可以站起来冲杯咖啡了。

下面配上一些choco常用的指令:

依次安装多个应用
```cmd
choco install <package1 package2 package3...>
```

安装指定版本的应用
```cmd
choco install foo -version 7.22.0
```

查看本地已安装应用
```cmd
choco list -localonly
```
简写
```cmd
choco list -lo
```

升级已安装应用
```cmd
choco  upgrade <package>
```

查看应用是否有新版本
```cmd
choco upgrade <package> --noop
```

查看chocolatey自身是否有新版本
```cmd
choco version chocolatey --noop
```

卸载应用
```cmd
choco uninstall <package>
```

查找应用
```cmd
choco search foo -all
```
或
```cmd
choco list foo -all
```

win中在cmd中刷新环境变量
```cmd
refreshenv
```

使用chocolatey基本上可以满足我们的大部分需求了，当然前提是网络流程，choco是依赖和服务器的通讯来查找最新的应用的。
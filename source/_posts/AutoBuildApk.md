---
title: Linux开发环境中使用shell/python脚本快速自动打包并查看apk文件
date: 2016-09-22 16:16:15
tags: [Android, 干货, python]
category: Android
---

最近工作中突然遇到一个很烦人的问题, 事情的起因是这样的. 我一个人开发/维护着大概5个项目, 负责服务端后台的同事经常在自己的本地电脑启着本地服务测试一些东西, 而他们的本地局域网IP是经常变动的, 那么问题来了, 只要他们的IP地址变动了就会过来找我针对某个局域网IP地址为某个项目打个包, 烦不胜烦. 甚至有些情况下需要两项目同时开发, AndroidStudio需要连着开好几个, 但是电脑受不了啊, 而且AndroidStudio有一个非常不友好的地方: 如果开启多个AndroidStudio工程, 只要某一个窗口卡了, 那么其余的工程界面也全都是无响应状态的. 当这两个问题重叠到一起的时候, 好了, 可用把鼠标和键盘放一边慢慢等着打包吧, 完了才能继续快乐的编程. 这简直不能忍.
今天针对这个问题, 记录一种解决方法, 实现在不打开AndroidStudio的情况下自动打包, 并在打包完成后打开apk所在文件夹.
```shell
./buildapk.sh
```
<!-- more -->

在AndroidStudio给项目打包的时候需要指定keystore然后输入密码,然后选keyAlias,再输入密码,然后才正正确的build以及签名.
那么如果需要自动打包,这些东西都不是我们想输入的,就应该怎么简单怎么来.
首先我们需要在gradle的配置文件中定义并指定release需要的签名文件以及密码等
```gradle
android {
    defaultConfig {
	...
    }
    signingConfigs {
        release {
            storeFile file('xxx.keystore')
            keyPassword 'xxx'
            keyAlias 'xxx'
            storePassword 'xxx'
        }
    }
    buildTypes {
        release {
            ...
            signingConfig signingConfigs.release
        }
    }
```

这样我们在使用gradle命令来进行build的时候就能够自动从配置文件中找到签名文件进行签名了
gradle的realease构建指令是这样的:
```sh
./gradlew assembleRelease
```
在项目的根目录下输入如上指令即可对项目进行构建并打包出apk并自动签名, 但是到这里对我们来说仍然不够, 在我的场景下打包出apk之后还要把这个文件发给同事, 这时候最起码要把打出来的apk让我能找得到.
在ubuntu系统下, 可用通过nautilus指令来用文件管理打开某个路径或某个文件所在的位置
```sh
nautilus nautilus ./app/build/outputs/apk/app-release.apk
```
这里可用使用相对路径也可以使用绝对路径

在项目的根目录新建一个buildapk.sh文件
把上面两条指令放进去
```sh
./gradlew assembleRelease
#!/bin/bash
./gradlew assembleRelease
apkPath="./app/build/outputs/apk"
apkFile=$apkPath"/app-release.apk"
unzip -qo $apkFile -d $apkPath"/temp"
keytool -printcert -file $apkPath"/temp/META-INF/CERT.RSA"
nautilus ./app/build/outputs/apk/app-release.apk
```
当我们要打包的时候进入项目的根目录执行这个脚本文件即可
```sh
./buildapk.sh
```
到此为止我们的shell脚本可以正常的自动打包并打印出签名信息, 但我们仍需要另外一些信息, 比如包名/版本号/应用名等
google给我提供的build-tools中有一个叫做aapt的工具给我们提供了这个功能, 使用这个工具类可以打印出apk文件的所有权限以及包的信息, 只需要从这些信息中抓取我们需要的即可
鉴于shell脚本的正则过滤不是很好用, 我们这里使用python来代理shell脚本, 并使用python的正则来进行正则校验
新建buildapk.py文件, 使用python的os模块来执行以及获取执行的打印结果
```python
import os,re
import subprocess

os.system("./gradlew assembleRelease")
apkPath = "./app/build/outputs/apk"
apkFile = apkPath + "/app-release.apk"
os.popen("unzip -qo "+apkFile+" -d "+apkPath+"/temp")
print os.popen("keytool -printcert -file " + apkPath + "/temp/META-INF/CERT.RSA").read()
```
这样我们上面的shell脚本就转化为python脚本了, 输入`python buildapk.py`执行脚本, 我们将得到和之前shell脚本同样的结果

接下来我们需要使用aapt来获取apk的详细信息
aapt位于SDK的build-tools下, 调用的命令为`$ANDROID_HOME/build-tools/23.0.3/aapt dump badging apkFile`, 前提是已经配置好ANDROID_HOME环境变量了, 反之则需要协商SDK的绝对路径.
把上面的命令使用python脚本来执行, 并将打印结果输入到字符串变量中
```python
appInfo = os.popen("$ANDROID_HOME/build-tools/23.0.3/aapt dump badging " + apkFile).read()
```
然后我们就可以快乐的用python强大的正则模块来抓取数据了: 如
```python
package = re.match(r".* name='([^']+)'.*", appInfo, re.M|re.I)
print "package: " + package.group(1)
```
这里选取package/versionCode/versionName/buildversion/applicationlabel这些信息来抓取并打印展示

完整的python文件
```python
import os,re
import subprocess
if __name__=="__main__": 
	os.system("./gradlew assembleRelease")
	apkPath = "./app/build/outputs/apk"
	apkFile = apkPath + "/app-release.apk"
	os.popen("unzip -qo "+apkFile+" -d "+apkPath+"/temp")
	print os.popen("keytool -printcert -file " + apkPath + "/temp/META-INF/CERT.RSA").read()
	appInfo = os.popen("$ANDROID_HOME/build-tools/23.0.3/aapt dump badging " + apkFile).read()
	package = re.match(r".* name='([^']+)'.*", appInfo, re.M|re.I)
	print "package: " + package.group(1)
	versionCode = re.match(r".* versionCode='([^']+)'.*", appInfo, re.M|re.I)
	print "versionCode: " + versionCode.group(1)
	versionName = re.match(r".* versionName='([^']+)'.*", appInfo, re.M|re.I)
	print "versionName: " + versionName.group(1)
	platformBuildVersionName = re.match(r".* platformBuildVersionName='([^']+)'.*", appInfo, re.M|re.I)
	print "platformBuildVersionName: " + platformBuildVersionName.group(1)
	applicationLabel = re.match(r"[\s\S]*application: label='([^']+)'.*", appInfo, re.M|re.I)
	print "applicationLabel: " + applicationLabel.group(1)
	subprocess.Popen("nautilus ./app/build/outputs/apk/app-release.apk", shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
```

打包后的输出内容:
```sh
...
:app:zipalignRelease UP-TO-DATE
:app:assembleRelease

BUILD SUCCESSFUL

Total time: 3.255 secs
所有者: CN=xxx, OU=xxx, O=xxx, L=xxx, ST=xxx, C=china
发布者: CN=xxx, OU=xxx, O=xxx, L=xxx, ST=xxx, C=china
序列号: 4ee590e7
有效期开始日期: Mon Dec 12 13:28:07 CST 2011, 截止日期: Fri Dec 05 13:28:07 CST 2036
证书指纹:
	 MD5: 4A:55:A7:3F:92:6D:25:8A:44:3E:6A:7A:CC:xx:xx:xx
	 SHA1: 93:69:E2:98:F5:F9:EC:AA:AA:D4:B2:22:C4:A6:02:B7:54:xx:xx:xx
	 SHA256: DC:CC:7F:85:F4:03:78:62:C8:1E:3C:75:5B:E9:F7:38:A2:50:B9:0E:93:1A:DE:BC:E7:C1:66:DA:5F:xx:xx:xx
	 签名算法名称: SHA1withRSA
	 版本: 3

package: xx.xx.xx
versionCode: 1
versionName: 1.2.2
platformBuildVersionName: 6.0-2704002
applicationLabel: xxxx
```

再回到最初的需求, 我们想在没有修改业务代码需求的情况下, 只变更ip或者版本号进行自动打包.
到此只实现了自动打包, 现在仍需要编写这只版本号和ip的部分, 这里用版本号的设定作为示例

如果你的工程是一个标准的AndroidStudio工程, 那么版本号的配置比然是写在buid.gradle文件中的, 要实现版本号的设定只需要重写build.gradle文件中的versionName字段即可
而python的正则模块中也为我们提供了正则替换的方法
```python
re.sub(r'versionName "[^"]+"', 'rersionName "新版本"', txt)
```
我们只需要讲配置文件读取到字符串中, 并使用正则替换即可
```python
def setVersonName(version):
	f = open("./app/build.gradle", "r")
	txt = f.read()
	f.close()

	txt = re.sub(r'versionName "[^"]+"', 'versionName "'+version+'"', txt)
	f = open("./app/build.gradle", "w")
	f.write(txt)
	f.close()

```
那这个方法该如何调用呢? setVersonName("1.1.1") 这样码? 太low!
如上这样的调用方法需要在每次修改version的时候都要修改python文件, 偷懒是提升技术的动力..
我们想要一种更加便捷的方法来使用 如:
```sh
python buildapk.py -v 1.1.1
```
只需要在python文件中加上参数列表的获取即可
```python
if __name__=="__main__":
	import sys
	vIndex = -1
	if "-v" in sys.argv:
		vIndex = sys.argv.index("-v")
	if vIndex>0 and len(sys.argv)>vIndex+1 :
		print("v")
		setVersonName(sys.argv[vIndex+1])
	
	...
```
sys.argv中传入了命令行后的参数列表, 这里首先判断参数中是否存在-v, 若存在则尝试获取取-v的下一个参数作为version

ip地址的设定同理, 不过修改的是java文件, 这里不再做赘述了

接下来的工作中就可以尽情的使用python脚本来构建apk了
某个ip来一个
```sh
python buildapk.py -i 192.168.x.x
```
某个ip来一个旧版本号的
```sh
python buildapk.py -i 192.168.x.x -v 1.1.0
```
正式服务器最新版的来一个
```sh
python buildapk.py -i www.xxx.com
```

完工.


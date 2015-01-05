---
layout: post
title: "goagent ubuntu"
date: 2015-01-05 18:59:57 +0800
comments: true
categories: 
---
Ubuntu下安装goagent
一生之盟 一生之盟 2013-06-25 12:37:22
下载并安装goagent和Google AppEngine SDK

goagent下载地址：http://code.google.com/p/goagent/

首页已经给出下载版本，本次使用goagent 1.7.10 稳定版。

Google AppEngine
SDK下载地址：https://code.google.com/intl/zh-CN/appengine/downloads.html

Google AppEngine SDK请下载Google AppEngine SDK for Python版本（linux）。

下载 Google AppEngine
SDK后解压google_appengine到自己的主目录，比如我的主目录是/home/leo/，解压完成后，进入/home/leo/google_appengine/。

下载goagent解压goagent到google_appengine目录下，解压完成应该存在/home/leo/google_appengine/goagent

配置并上传goagent。

打开goagent目录，进入目录下server/python目录，修改app.yaml文件中application为刚刚申请的AppID。

在google_appengine目录下，执行./appcfg.py update
./goagent/server/python上传，会提示输入AppID,以及你的邮箱，和刚申请的应用程序专用密码。

在goagent目录下，下该local目录下proxy.ini文件，将文件中[ gae  ]项目的appid修改为
你使用的AppID.

然后在local目录下，运行python proxy.py，即可使用代理。

安装SwitchySharp插件，最后导入这个设置http://goagent.googlecode.com/files/SwitchyOptions.bak导入文件是指在SwitchySharp插件选项中的导入/导出设置，从文件恢复。

安装证书导入工具

apt-get install libnss3-tools 

将goAgent文件夹内的证书文件CA.crt导入(注意证书的绝对路)

    certutil -d sql:$HOME/.pki/nssdb -A -t TC -n "goagent" -i
    /home/dn/google_appengine/goagent-goagent-91cd5e4/local/CA.crt  


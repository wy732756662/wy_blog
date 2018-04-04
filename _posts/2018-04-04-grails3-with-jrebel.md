---
layout: post
title:  "Grails 3 with JRebel"
date:   2018-4-4 12:13:10 +0800
categories: grails
tag: JRebel
---

JRebel版本：2018.1.0



为何使用JRebel：

    Grails在每个版本里都会自带一些插件，比如自动集成Spring、Hibernate等，而Spring load也是每个版本都自带的插件，但这个功能非常鸡肋、并且难以使用，想想在Grails2一些列版本里，修改service层没有问题，但是一旦修改的domain层代码就立即崩溃，需要重新启动系统，而Grails 3以后使用的是Gradle编译，Gradle编译的速度有大家公认的慢（Grails2使用的是Gank，速度还可以接受），所以这里引入JRebel插件进行热编译（即使修改domain也不需要重启）。使用Grails和JRebel的介绍：https://zeroturnaround.com/rebellabs/get-groovy-with-jrebel-and-grails



IDEA集成：



1、安装最新版JRebel

    选择File->Settings->Plugins，在搜索框搜索JRebel

<p align="center">
  <img src="/wy_blog/pic/JRebel/1.png" width="700px"/>
</p>


点击Search in repositories，在弹窗的窗口右侧选择install（安装完后需要重启IDEA）



2、激活JRebel

    由于JRebel是从7.1.7开始支持的Grails3.3.2，并且目前网上还没有办法破解该版本，所以先试用14天（试用期过了有办法再次试用，不过每次操作可能花费10分钟时间）。

    试用方法：去JRebel官网注册一个账号，地址：https://zeroturnaround.com/software/jrebel/trial/  ，在右侧填好信息后（信息随便填即可），点击注册，成功后会返回一个激活码。

    在IDEA里点击Help->JRebel->Activation，在弹出的窗口上面选择I already have a license

<p align="center">
  <img src="/wy_blog/pic/JRebel/2.png" width="700px"/>
</p>

然后将之前拿到的激活码输入，然后点击Activate JRebel。



3、禁用spring load

    打开主项目下的build.gradle，具体代码：


<code>
    grails{
        //禁用spring load
        agent {
            enabled = false
        }
    }
</code>


4、使用JRebel

    点击IDEA里的View->Tool Windows->JRebel，在弹出的窗口里勾选第一排里所有以_ast结尾的选项

<p align="center">
  <img src="/wy_blog/pic/JRebel/3.png" width="700px"/>
</p>


点击右上角新的Debug按钮运行项目

<p align="center">
  <img src="/wy_blog/pic/JRebel/4.png" width="700px"/>
</p>


##### [jekyll]:      [http://jekyllrb.com](http://jekyllrb.com)

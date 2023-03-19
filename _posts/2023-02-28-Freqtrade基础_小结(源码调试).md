---
date: 2023-02-28
title: Freqtrade基础_小结(源码调试)
categories:
  - knowledge
  - freqtrade
author_staff_member: dumengru
---

我们已经简单走完了文件配置, 策略编写, 策略回测及策略仿真的全部步骤, 基础内容暂时告一段落, 今天分享源码调试部分内容

## 文件介绍

![]({{site.baseurl}}/images/202302231411.png)

打开Freqtrade项目目录

1. 输入命令的执行逻辑都存放在"commands"目录中

2. Freqtrade程序入口文件是"\__main__.py"

3. 机器人交易入口文件是"worker.py", 

4. 机器人交易主逻辑文件是"freqtradebot.py"

5. 机器人回测主逻辑文件是"optimize/backtesting.py"

6. 机器人策略模板文件是"strategy/interface.py"

## 断点调试(pycharm)

1. 找到"__main__.py"文件打上断点

2. 点击Pycharm中"Run" -> "Edit Configuration"
![]({{site.baseurl}}/images/202302231424.png)

3. 在"Parameters"中输入Freqtrade的命令

![]({{site.baseurl}}/images/202302231427.png)

如我这里的输入对应在cmd中的命令
```python
freqtrade backtesting --timeframe 5m --strategy SampleStrategy --userdir ../user_data --timerange=20230101-
```

注意1: 不需要"freqtrade"这个词

注意2: userdir的相对路径是相对于__main__.py这个文件的, 注意填写正确

4. 以debug方式运行__main__.py文件即可调试上述命令内容

## 命令入口

打开"commands/arguments.py"文件, 在210行之后写着所有的命令入口函数

![]({{site.baseurl}}/images/202302231444.png)

如我们按住键盘"Ctrl"并点击"start_trading"函数, 就会进入"commands/trade_commands.py"文件, 这里便是启动机器人的代码

![]({{site.baseurl}}/images/202302231448.png)

可以看到启动机器人的代码很简单, 传入参数(命令中输入的参数, 配置文件中的参数及一些默认参数), 然后创建了Worker对象, 最后执行`worker.run()`便结束了. 要想进一步查看逻辑, 感兴趣可以继续追踪下去

## 总结

不管是查阅Freqtrade的官方文档或代码示例, 亦或是搜索网络文章, 难免有些疏漏或不详尽的地方需要我们自己探究

**talk is cheap, show me the code**
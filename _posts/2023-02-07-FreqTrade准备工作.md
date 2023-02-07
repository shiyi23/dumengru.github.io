---
date: 2023-02-07
title: FreqTrade准备工作
categories:
  - tool
  - freqtrade
author_staff_member: dumengru
---

本文主要介绍FreqTrade的详细安装步骤及简单使用.

## 安装

1. 先安装TA-Lib
2. 将Github上"freqtrade"项目下载到本地
3. 将"freqtrade/requirements.txt"文件中涉及"TA-Lib"那行注释掉(因为项目要求的版本和已安装的版本可能不一样)
3. 打开cmd, 进入"freqtrade"目录输入以下命令(安装项目所需所有工具包)
```python
pip install -r requirements.txt
```
4. 继续输入以下命令(安装freqtrade命令工具)
```python
pip install -e .
```
5. 测试安装成功输入(不出现"不是内部或外部命令..."即为成功)
```python
freqtrade
```
![]({{site.baseurl}}/images/202302070003.png)

## 常用命令

"freqtrade"项目很规范, 所有的数据和策略都存放在一个文件夹下, 默认是"freqtrade/user_data/", 为了不混淆文件, 我新建了一个空目录"freqtrade_test", 以下所有的操作都在该目录下进行.

**新建工作目录**

"freqtrade_test"一开始是空目录, 输入以下命令会自动出现一些默认文件(将当前目录设为工作目录):
```shell
freqtrade create-userdir --userdir ./
```
![]({{site.baseurl}}/images/202302070023.png)

**新建配置**

在当前目录新建配置文件
```shell
freqtrade new-config
```
![]({{site.baseurl}}/images/202302072044.png)

**新建策略**

在当前目录新建名为"strategy_test"的模板策略(默认放在strategies文件夹下)
```python
freqtrade new-strategy --strategy strategy_test --userdir ./
```

**查看策略**

查看当前路径"strategies"目录下的策略
```python
freqtrade list-strategies --userdir ./
```
![]({{site.baseurl}}/images/202302072053.png)

**查看交易所**

查看FreqTrade支持的交易所
```shell
freqtrade list-exchanges
```
![]({{site.baseurl}}/images/202302072055.png)

**查看时间周期**

查看binance交易所支持交易的时间周期

```shell
freqtrade list-timeframes --exchange binance --userdir ./
```

**查看货币对**

查看binance交易所支持交易的货币对

```shell
freqtrade list-markets --exchange binance --userdir ./
```

![]({{site.baseurl}}/images/202302072101.png)

**数据下载**

下载binance交易所从20230201开始到现在的5分钟BTC/USDT数据, 下载完会自动保存到"data"目录下

```shell
freqtrade download-data --exchange binance --pairs BTC/USDT --userdir ./ --timeframe 5m --timerange 20230201-
```

## 总结

以上命令都可以从官方文档中找到更详细的描述, 还有更多其他命令今后遇到时再提及.

仅仅几句简单的命令, 就完成了很多量化交易前期的准备工作. 你是否摩拳擦掌, 跃跃欲试呢?

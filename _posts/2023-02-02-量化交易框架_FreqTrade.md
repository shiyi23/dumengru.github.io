---
date: 2023-02-02
title: 量化交易框架_Freqtrade
categories:
  - tool
  - freqtrade
author_staff_member: dumengru
---

CCXT只是统一了各个交易所的接口, 但是要实现一个完整的量化交易策略, 仅仅实现"可交易"是不够的, Freqtrade项目构建了一个更加完善的量化交易系统. 

## 简介

Freqtrade是一个用Python编写的免费开源的加密货币交易机器人, 它包含回测, 绘图和资金管理工具以及通过机器学习进行的策略优化.

Github地址: https://github.com/freqtrade/freqtrade

文档地址: https://www.freqtrade.io/en/stable/

## 优点

**文档详细**

Freqtrade的所有相关功能用法基本都能从文档中找到. (与此相对, 国内的很多开源项目在文档方面不够重视)

**功能完整**

包括策略编写, 回测, 仿真, 实盘, 参数优化, 数据下载, 投研分析等量化交易相关几乎所有功能.

同时还提供了可视化的界面以及通过Telegram控制自己的交易机器人.
![]({{site.baseurl}}/images/202302020031.png)

**简单高效**

所有功能都可以使用一句命令实现.

#### 缺点

**策略逻辑**

使用Freqtrade编写策略是"并行"而非"线性"的.

举例而言, 当我们使用VeighNa或WonderTrader编写策略时, 可以将数据想象成"逐条回放", 策略逻辑也在逐条回放的逻辑中顺序编写; 但是使用Freqtrade编写策略时, 所有数据同时存放在一个DataFrame(二维表)中, 策略逻辑必须使用pandas实现. (应该是为了提高回测速度, 但确实有点反人性)

**最后用一句话作为文章的总结: 作为一个免费开源的量化交易机器人项目, Freqtrade足够完美**
---
date: 2023-01-19
title: 量化交易框架_WonderTrader
categories:
  - tool
  - wtcpp
  - wtpy
author_staff_member: dumengru
---

前几日, 在软件测试公司Tiobe公布的编程语言排行榜中, C++被评选为2022年度最佳编程语言.

上一次C++被评为第一还是在2003年, 正应了那句老话: **它大爷终究是它大爷**.

![]({{site.baseurl}}/images/202301192154.png)

Tiobe排行榜地址: https://www.tiobe.com/tiobe-index/

在国内的量化交易领域, 尽管Python一路高歌猛进, 攻城拔寨, 但C++的霸主地位依旧无可替代.

有这么一个开源的量化交易框架, 它的核心架构使用C++编写, 同时提供了Python的应用接口. 即兼顾了C++的执行效率, 又包容了Python易学易用, 它就是**WonderTrader**

## 简介

WonderTrader是一个基于C++核心模块的, 适应全市场全品种交易的, 高效率, 高可用的量化交易开发框架.

WonderTrader提供的Python应用框架被独立出来, 称为wtpy(官方), 与此对应, 以后我们统称其C++框架为wtcpp(非官方).

GitHub地址(wtcpp): https://github.com/wondertrader/wondertrader

Gitee地址(wtcpp): https://gitee.com/wondertrader/wondertrader

GitHub地址(wtpy): https://github.com/wondertrader/wtpy

Gitee地址(wtpy): https://gitee.com/wondertrader/wtpy

知乎地址: https://www.zhihu.com/people/yanguoye/posts

文档地址(官方): https://wondertrader.github.io/#/

文档地址(粉丝): https://dumengru.com/docs_wondertrader/

## 优势

**编程语言**

C++的编程语言优势得天独厚.

**实盘检验**

项目开发者是某上亿规模的私募CTO, 因此WonderTrader经历了数十亿级实盘管理规模的检验.

**交易引擎**

![]({{site.baseurl}}/images/202301192158.png)
1. 高频策略引擎(CTA引擎), 事件+时间驱动, 适用于CTA策略.
2. 异步策略引擎(SEL引擎), 时间驱动, 适用于多因子选股策略.
3. 高频策略引擎(HFT引擎), 事件驱动, 系统延迟在1-2微秒之间.
4. 极速策略引擎(UFT引擎), 事件驱动, 系统延迟在200纳秒之内.

**组合管理**

轻松处理多策略组合管理, 目标仓位合并避免自成交, 多账户并发执行.

**风控管理**

1. 资金风控: 控制资金回测.
2. 合规风控: 避免频繁下单撤单.
3. 离合器机制: 断开信号执行, 可以继续观察策略表现.

## 劣势

**编程语言**

C++的编程语言劣势与生俱来. (如果不想搞清楚底层运行逻辑, 也可以闭着眼睛使用wtpy)

**文档说明**

没有较为详细的使用说明文档, 上手难度较大.(半官方文档, 笔者尽力更新...)

**配置文件复杂**

从使用角度看, WonderTrader明显是**配置驱动型**的, 大量的配置文件让人眼花缭乱.

公共配置
![公共配置]({{site.baseurl}}/images/202301192200.png)
CTA策略配置
![公共配置]({{site.baseurl}}/images/202301192201.png)

**个人项目**

WonderTrader目前是个人开发的项目, 将来会不会突然不开源, 不维护, 突然开始收费...都是有可能发生的.

**最后用两句话作为文章的总结: WonderTrader确实上手难度较大, 但也足够专业. 欲戴王冠, 必承其重.**
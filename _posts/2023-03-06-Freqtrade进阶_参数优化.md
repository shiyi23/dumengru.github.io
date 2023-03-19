---
date: 2023-03-06
title: Freqtrade进阶_参数优化
categories:
  - knowledge
  - freqtrade
author_staff_member: dumengru
---

在编写策略时我们可能会设置很多参数, 最常见的便是各类技术指标参数. 参数值具体设置多少比较合适需要根据参数优化的结果判断

## 简述

#### 穷举法

最普通的参数优化方案便是将不同参数组合以穷举方式列出, 然后逐个回测. VeighNa, WonderTrader, MetaTrade5都有此功能, 但是这是一种最不常用的参数优化方案

想象一下假设我的一个策略中有5个参数, 每隔参数有20个备选值, 那么穷举则共有20**5=320万种组合, 如果回测1次需要1分钟, 那么...

#### 遗传算法

还有一种比较常见的优化方案是"遗传算法"优化, VeighNa, WonderTrader, MetaTrade5也提供此种优化方式, 这种方式能够大大缩小参数的备选组合范围, 从而节省大量时间, 因此也是上述框架最常用的优化方案

#### 机器学习算法

Freqtrade提供了一种更加高级的参数优化方式, 它使用贝叶斯搜索和机器学习中的回归算法(目前是ExtraTreesRegressor)能够快速搜索超参数空间中的参数组合, 使得损失函数的值最小(我们的回测结果最优)

从性能上讲, 这种方式和遗传算法区别不大, 但是为什么说它"高级"呢? 因此这种方式可以自定义迭代次数. 假设我们有320万组备选的超参数空间, 每次回测需要1分钟, 遗传算法压缩到极致需要迭代10000次, 那也一共需要10000分钟的时间. 但是如果使用Freqtrade指定迭代次数为300, 则只会使用300分钟

## 基本概念

#### 参数类型

Freqtrade提供了以下几种可优化的参数类型
- IntParameter: 整数类型
- DecimalParameter: 浮点数类型, 默认精度为3
- RealParameter: 高精度浮点类型, 几乎不用(使用DecimalParameter即可) 
 -CategoricalParameter: 自定义列表类型
 - BooleanParameter: 布尔类型, 可用"CategoricalParameter([True, False])"替代

#### 优化空间

1. 进出场信号空间, 与参数类型组合使用
- 入场信号: 在参数类型中添加参数"space='buy'"或者参数名以"buy_"开头
- 离场信号: 在参数类型中添加参数"space='sell'"或者参数名以"sell_"开头

2. roi_space: 优化ROI
3. generate_roi_table: 自定义优化ROI
4. stoploss_space: 优化止损
5. trailing_space: 优化跟踪止损
6. max_open_trades_space: 优化max_open_trades

#### 使用示例

```python
class MyAwesomeStrategy(IStrategy):
    # 优化buy_adx, 范围20-40, 步长值为0.1(精度为1), 默认值30.1, 属于入场信号空间
    buy_adx = DecimalParameter(20, 40, decimals=1, default=30.1, space="buy")
    # 优化buy_rsi, 范围20-40, 步长值为1(默认), 默认值30, 属于入场信号空间
    buy_rsi = IntParameter(20, 40, default=30, space="buy")
    # 优化buy_adx_enabled, 范围True/False, 默认值True, 属于入场信号空间
    buy_adx_enabled = BooleanParameter(default=True, space="buy")
    # 优化buy_rsi_enabled, 范围True/False, 默认值False, 属于入场信号空间
    buy_rsi_enabled = CategoricalParameter([True, False], default=False, space="buy")
    # 优化buy_trigger, 范围bb_lower/macd_cross_signal, 默认值bb_lower, 属于入场信号空间
    buy_trigger = CategoricalParameter(["bb_lower", "macd_cross_signal"], default="bb_lower", space="buy")
```

输入参数优化指令指令, 选择优化空间roi, stoploss和buy, 最大迭代100次

```python
freqtrade hyperopt --hyperopt-loss SharpeHyperOptLossDaily --spaces roi stoploss buy --strategy MyWorkingStrategy --config config.json -e 100
```

![]({{site.baseurl}}/images/202302222136.png)

#### 指标数据

由于回测中会先计算指标数据, 因此为了参数优化过程顺利进行, 首先需要在"populate_indicators"方法中将我们需要的指标数据全部计算出来

```python
from pandas import DataFrame
from functools import reduce
import talib.abstract as ta

from freqtrade.strategy import (BooleanParameter, CategoricalParameter, DecimalParameter, IStrategy, IntParameter)
import freqtrade.vendor.qtpylib.indicators as qtpylib

class MyAwesomeStrategy(IStrategy):
    stoploss = -0.05
    timeframe = '15m'
    minimal_roi = {
        "0":  0.10
    }
    # 设置入场参数优化空间
    buy_ema_short = IntParameter(3, 50, default=5)
    buy_ema_long = IntParameter(15, 200, default=50)

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 遍历buy_ema_short参数空间, 并计算指标值
        for val in self.buy_ema_short.range:
            dataframe[f'ema_short_{val}'] = ta.EMA(dataframe, timeperiod=val)

        # 遍历buy_ema_long参数空间, 并计算指标值
        for val in self.buy_ema_long.range:
            dataframe[f'ema_long_{val}'] = ta.EMA(dataframe, timeperiod=val)

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        conditions = []
        # 计算参数值对应的buy_ema_short上穿buy_ema_long信号
        conditions.append(qtpylib.crossed_above(
                dataframe[f'ema_short_{self.buy_ema_short.value}'], dataframe[f'ema_long_{self.buy_ema_long.value}']
            ))
        conditions.append(dataframe['volume'] > 0)
        # 满足上穿条件将入场信号设为1
        if conditions:
            dataframe.loc[
                reduce(lambda x, y: x & y, conditions),
                'enter_long'] = 1
        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        conditions = []
        # 计算参数值对应的buy_ema_long上穿buy_ema_short信号
        conditions.append(qtpylib.crossed_above(
                dataframe[f'ema_long_{self.buy_ema_long.value}'], dataframe[f'ema_short_{self.buy_ema_short.value}']
            ))
      
        conditions.append(dataframe['volume'] > 0)
        # 满足上穿条件将离场信号设为1
        if conditions:
            dataframe.loc[
                reduce(lambda x, y: x & y, conditions),
                'exit_long'] = 1
        return dataframe
```

## 其他优化

#### 优化测类保护

Freqtrade还可以优化策略保护的参数, 需要定义特殊的参数空间"protection"

```python
from pandas import DataFrame
from functools import reduce

import talib.abstract as ta

from freqtrade.strategy import (BooleanParameter, CategoricalParameter, DecimalParameter, IStrategy, IntParameter)
import freqtrade.vendor.qtpylib.indicators as qtpylib

class MyAwesomeStrategy(IStrategy):
    stoploss = -0.05
    timeframe = '15m'
    # 定义参数优化空间, protection
    cooldown_lookback = IntParameter(2, 48, default=5, space="protection", optimize=True)
    stop_duration = IntParameter(12, 200, default=5, space="protection", optimize=True)
    use_stop_protection = BooleanParameter(default=True, space="protection", optimize=True)

    @property
    def protections(self):
        """ 测类保护逻辑 """ 
        prot = []
        # 在卖出后停止 cooldown_lookback 根K线交易
        prot.append({
            "method": "CooldownPeriod",
            "stop_duration_candles": self.cooldown_lookback.value
        })
        # 如果启用了 StoplossGuard
        if self.use_stop_protection.value:
            # 24*3根K线内超过4次止损则停止交易stop_duration根K线
            prot.append({
                "method": "StoplossGuard",
                "lookback_period_candles": 24 * 3,
                "trade_limit": 4,
                "stop_duration_candles": self.stop_duration.value,
                "only_per_pair": False
            })
        return prot
```

执行优化命令
```python
freqtrade hyperopt --hyperopt-loss SharpeHyperOptLossDaily --strategy MyAwesomeStrategy --spaces protection
```

#### 优化max_entry_position_adjustment

```python
from pandas import DataFrame
from functools import reduce

import talib.abstract as ta

from freqtrade.strategy import (BooleanParameter, CategoricalParameter, DecimalParameter, 
                                IStrategy, IntParameter)
import freqtrade.vendor.qtpylib.indicators as qtpylib

class MyAwesomeStrategy(IStrategy):
    stoploss = -0.05
    timeframe = '15m'
    # 定义入场信号空间
    max_epa = CategoricalParameter([-1, 0, 1, 3, 5, 10], default=1, space="buy", optimize=True)

    @property
    def max_entry_position_adjustment(self):
        return self.max_epa.value
```

## 损失函数

Freqtrade提供了以下几种损失函数类型:

- ShortTradeDurHyperOptLoss: Freqtrade默认的损失函数, 主要是为了缩短交易时间并避免损失
- OnlyProfitHyperOptLoss: 只考虑利润
- SharpeHyperOptLoss： 计算投资回报的夏普比率。
- SharpeHyperOptLossDaily: 计算每日投资回报的夏普比率
- SortinoHyperOptLoss: 计算投资回报的Sortino比率
- SortinoHyperOptLossDaily: 计算每日投资回报的Sortino比率。
- MaxDrawDownHyperOptLoss: 只考虑最大绝对回撤
- MaxDrawDownRelativeHyperOptLoss: 优化最大绝对回撤, 同时调整最大相对回撤
- CalmarHyperOptLoss: 计算投资回报的Calmar比率
- ProfitDrawDownHyperOptLoss: 考虑最大利润和最小亏损

指定优化目标的命令

```python
freqtrade hyperopt --config config.json --hyperopt-loss <hyperoptlossname> --strategy <strategyname> -e 500 --spaces all
```

## 指定优化空间

Freqtrade优化命令可以指定以下默认空间

- all: 所有参数空间
- buy: 定义为buy的参数空间
- sell: 定义为sell的参数空间
- roi: ROI空间
- stoploss: 搜索最佳止损值
- trailing: 搜索最佳追踪止损值
- trades: 搜索最佳最大未平仓交易值
- protection: 搜索最佳保护参数
- default: all除了trailing和protection

## 注意事项

Freqtrade默认每个货币对只允许1个交易, 所有货币对的交易总数也受到"max_open_trades"限制

"--enable-position-stacking" 可以允许模拟多次购买同一对

"--disable-max-market-positions" 可以禁用"max_open_trades"(相当于"max_open_trades"设置了非常高的数值)

## 总结

参数优化是策略编写完成之后必不可少的一个环节, 随着硬件技术的提高和算法的优化, 我们也许可以在很短的时间内从上百万组参数中快速筛选出最优结果, 使得原本亏损的策略"脱胎换骨", 但是, 切忌进入参数优化陷阱: 过拟合

参数优化只是辅助手段, 策略逻辑才是赢利的核心

**garbage in, garbage out**

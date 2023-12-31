---
date: 2023-02-22
title: Freqtrade基础_回调函数
categories:
  - knowledge
  - freqtrade
author_staff_member: dumengru
---

不管是VeighNa, WonderTrader, MT5还是Freqtrade, 只要了解了它们的策略函数回调逻辑, 理论上我们就可以用其编写量化交易策略了.

## 策略编写逻辑

VeighNa中有一个策略模板类叫CtaTemplate, WonderTrader中有一个策略模板类叫CtaStrategy, 在用它们编写CTA策略时, 只需要将自己的策略类继承模板类, 然后重写特定的方法即可.

![]({{site.baseurl}}/images/202302211618.png)

Freqtrade中的策略模板类在"freqtrade/strategy/interface.py"文件中, 叫**IStrategy**, 因此在我们编写策略时只需要继承它, 并通过重写它的方法从而实现我们的交易逻辑即可. 

所有的方法中有只有唯一一个被"abstractmethod"装饰器装饰的方法叫"populate_indicators", 意味着子类必须重写该方法. 因此, 从理论上讲, 使用Freqtrade编写最简单的策略只需要重写"populate_indicators"即可(这个方法用来计算技术指标, 开平仓可以通过风控配置实现).

## 回调函数

回测和实盘的函数回调流程在官方文档里也很详细: https://www.freqtrade.io/en/stable/bot-basics/

以下是Freqtrade编写策略涉及到的相关方法:

#### bot_start

策略开始时执行1次

#### bot_loop_start

- 回测: 策略开始时执行1次
- 实盘: 每隔N秒执行1次, 通过"internals.process_throttle_secs"配置设置

#### populate_indicators
返回计算好指标的DataFrame(pandas数据类型)

#### populate_entry_trend
返回计算好入场信号的DataFrame(pandas数据类型)

#### populate_exit_trend
返回计算好离场信号的DataFrame(pandas数据类型)

#### custom_stake_amount

返回浮点数, 即自定义下单量. 返回0或空值则禁止开仓. 

#### custom_exit

返回非空值string或True相当于设置了退出信号.

**注意事项**

1. 如果设置参数"use_exit_signal=False", 该方法不会调用
2. 如果启用该方法, 将忽略"exit_profit_only"配置

#### custom_stoploss

返回一个百分数, 即当前价格的百分比作为止损. 假设当前价格为100, 返回0.04即将止损设置为96.

**注意事项**

1. 必须在策略中添加"use_custom_stoploss = True", 该方法才能被调用
2. 百分数忽略正负号, 即返回0.04和-0.04结果一致
3. 止损价格只能向上移动, 即若当前计算出新的止损价小于之前的止损价则会被忽略
4. 返回1使用默认止损

#### custom_entry_price/custom_exit_price

返回浮点数, 即自定义的入场和离场价格. 返回无效值则使用默认价格

**注意事项**

该方法和"custom_price_max_distance_ratio"相关, "custom_price_max_distance_ratio"默认为0.02, 若当前价格为100, 则下单价格范围应该在98-102, 若该方法返回97, 则会被忽略

#### check_entry_timeout/check_exit_timeout

返回True则取消订单, 返回False则保持订单有效

#### confirm_trade_entry/confirm_trade_exit
在交易确认的最后一步, 返回True确认交易, 返回False取消交易

#### adjust_trade_position
返回浮点数即调整的下单量, 返回空值不做处理

**注意事项**

1. 必须设置"position_adjustment_enable=True", 启用该方法
2. "max_entry_position_adjustment"参数限制机器人可执行的额外下单量, 默认-1, 即无限制
3. 存在有等待执行的买入或卖出订单时, 不会调用此方法

#### adjust_entry_price

返回浮点数即修改的入场价, 返回空值不做处理

#### leverage

返回浮点数即设置的交易杠杆, 默认为1即无杠杆

**注意事项**
1. 是否可以使用杠杆需要根据自己选择的交易所确定
2. 所有的利润计算, 止损止盈计算都包括杠杆, 假设在10倍杠杆中使用10%的止损, 则价格逆向移动1%即会被止损

## 特殊用法示例

#### 自定义止损

**时间跟踪止损**

前60分钟使用默认止损, 之后使用从最高点回撤10%的跟踪止损, 120分钟后使用从最高点回撤5%的跟踪止损.

```python
from datetime import datetime, timedelta
from freqtrade.persistence import Trade

class AwesomeStrategy(IStrategy):
    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: 'Trade', current_time: datetime,
                        current_rate: float, current_profit: float, **kwargs) -> float:

        # Make sure you have the longest interval first - these conditions are evaluated from top to bottom.
        if current_time - timedelta(minutes=120) > trade.open_date_utc:
            return -0.05
        elif current_time - timedelta(minutes=60) > trade.open_date_utc:
            return -0.10
        return 1
```

**货币对跟踪止损**

根据不同货币对设置不同跟踪止损

```python
from datetime import datetime
from freqtrade.persistence import Trade

class AwesomeStrategy(IStrategy):
    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: 'Trade', current_time: datetime,
                        current_rate: float, current_profit: float, **kwargs) -> float:

        if pair in ('ETH/BTC', 'XRP/BTC'):
            return -0.10
        elif pair in ('LTC/BTC'):
            return -0.05
        return -0.15
```

**利润跟踪止损**

开始使用默认止损, 当利润超过4%时, 使用当前利润的50%跟踪止损, 且最低为2.5, 最高为5%

```python
from datetime import datetime, timedelta
from freqtrade.persistence import Trade

class AwesomeStrategy(IStrategy):
    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: 'Trade', current_time: datetime,
                        current_rate: float, current_profit: float, **kwargs) -> float:

        if current_profit < 0.04:
            return -1 # return a value bigger than the initial stoploss to keep using the initial stoploss

        # After reaching the desired offset, allow the stoploss to trail by half the profit
        desired_stoploss = current_profit / 2

        # Use a minimum of 2.5% and a maximum of 5%
        return max(min(desired_stoploss, 0.05), 0.025)
```

**阶梯式止损**

开始使用默认止损, 利润大于20%止损设置高于开盘价7%, 利润大于25%止损设置高于开盘价15%, 利润大于40%止损设置高于开盘价25%

"stoploss_from_open"是Freqtrade自带的策略辅助函数

```python
from datetime import datetime
from freqtrade.persistence import Trade
from freqtrade.strategy import stoploss_from_open

class AwesomeStrategy(IStrategy):
    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: 'Trade', current_time: datetime,
                        current_rate: float, current_profit: float, **kwargs) -> float:

        # evaluate highest to lowest, so that highest possible stop is used
        if current_profit > 0.40:
            return stoploss_from_open(0.25, current_profit, is_short=trade.is_short)
        elif current_profit > 0.25:
            return stoploss_from_open(0.15, current_profit, is_short=trade.is_short)
        elif current_profit > 0.20:
            return stoploss_from_open(0.07, current_profit, is_short=trade.is_short)

        # return maximum stoploss value, keeping current stoploss price unchanged
        return 1
```

**指标止损**

使用低于当前价格的sar指标值作为止损

```python
class AwesomeStrategy(IStrategy):
    
    # 计算sar指标
    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['sar'] = ta.SAR(dataframe)

    use_custom_stoploss = True
    def custom_stoploss(self, pair: str, trade: 'Trade', current_time: datetime,
                        current_rate: float, current_profit: float, **kwargs) -> float:

        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()

        # Use parabolic sar as absolute stoploss price
        stoploss_price = last_candle['sar']

        # Convert absolute price to percentage relative to current_rate
        if stoploss_price < current_rate:
            return (stoploss_price / current_rate) - 1

        # return maximum stoploss value, keeping current stoploss price unchanged
        return 1
```

#### 自定义订单超时示例

对价格较大的货币对限制较短超时撤单, 对价格较小的货币对限制较长时间超时撤单

```python
from datetime import datetime, timedelta
from freqtrade.persistence import Trade, Order

class AwesomeStrategy(IStrategy):
    # 设置订单默认25小时之后取消
    unfilledtimeout = {
        'entry': 60 * 25,
        'exit': 60 * 25
    }

    def check_entry_timeout(self, pair: str, trade: 'Trade', order: 'Order',
                            current_time: datetime, **kwargs) -> bool:
        if trade.open_rate > 100 and trade.open_date_utc < current_time - timedelta(minutes=5):
            return True
        elif trade.open_rate > 10 and trade.open_date_utc < current_time - timedelta(minutes=3):
            return True
        elif trade.open_rate < 1 and trade.open_date_utc < current_time - timedelta(hours=24):
           return True
        return False


    def check_exit_timeout(self, pair: str, trade: Trade, order: 'Order',
                           current_time: datetime, **kwargs) -> bool:
        if trade.open_rate > 100 and trade.open_date_utc < current_time - timedelta(minutes=5):
            return True
        elif trade.open_rate > 10 and trade.open_date_utc < current_time - timedelta(minutes=3):
            return True
        elif trade.open_rate < 1 and trade.open_date_utc < current_time - timedelta(hours=24):
           return True
        return False
```

## 总结

**如果说配置是策略编写的基础, 那么回调函数就是策略编写的灵魂**

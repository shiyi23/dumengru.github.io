---
date: 2023-02-23
title: Freqtrade基础_新建策略
categories:
  - knowledge
  - freqtrade
author_staff_member: dumengru
---

了解了配置文件和回调函数之后, 我们已经可以编写交易策略了.

## 准备工作
1. 新建工作目录, 命名为"user_data"

- 注意1: "enable Telegram"和"enable the Rest API"都选择No
- 注意2: stake_amount * max_open_trades < dry_run_wallet

```python
freqtrade create-userdir --userdir user_data
```

2. 进入"user_data"目录, 新建配置文件
```python
freqtrade new-config
```

3. 在当前目录下, 新建一个最简单的策略, 名为SampleStrategy

```python
freqtrade new-strategy --userdir ./ --strategy SampleStrategy --template minimal
```

## 策略解析

```python
class SampleStrategy(IStrategy):
    """ """
    # 策略版本, 最新版: 3
    INTERFACE_VERSION = 3
    # 策略的时间周期
    timeframe = '5m'
    # 策略是否可做空
    can_short: bool = False
    # 开仓30分钟内有4%的利润立即平仓, 之后有2%的利润立即平仓, 60分钟后有1%的利润立即平仓
    minimal_roi = {
        "60": 0.01,
        "30": 0.02,
        "0": 0.04
    }
    # 止损10%
    stoploss = -0.10
    # 不启用跟踪止损
    trailing_stop = False
    # 仅在出现新的K线执行一次策略逻辑, 即5分钟执行1次
    process_only_new_candles = True
    # 启用退出信号, 即可回调 custom_exit
    use_exit_signal = True
    # 不管是否有利润, 都可以退出
    exit_profit_only = False
    # 不管入场信号是否有效, 都要根据roi设置的止损/止盈退出
    ignore_roi_if_entry_signal = False
    # 策略开始前准备30根K线数据, 即5min*30=2.5h数据
    startup_candle_count: int = 30
    # 策略参数: rsi指标买入信号默认30, rsi指标卖出信号默认70
    buy_rsi = IntParameter(10, 40, default=30, space="buy")
    sell_rsi = IntParameter(60, 90, default=70, space="sell")
    # 订单类型: 入场和离场使用限价单, 止损使用市价单, 止损单不要放在交易所
    order_types = {
        'entry': 'limit',
        'exit': 'limit',
        'stoploss': 'market',
        'stoploss_on_exchange': False
    }
    # 订单过期类型: 一直有效直至取消
    order_time_in_force = {
        'entry': 'GTC',
        'exit': 'GTC'
    }

    def informative_pairs(self):
        """ """
        return []

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """ """
        # 计算rsi指标之后返回数据
        dataframe['rsi'] = ta.RSI(dataframe)
        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """ """
        # rsi指标值上穿30, 添加做多信号: "enter_long" = 1
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], self.buy_rsi.value)) &  # Signal: RSI crosses above buy_rsi
                (dataframe['volume'] > 0)  # Make sure Volume is not 0
            ),
            'enter_long'] = 1
        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """ """
        # rsi指标上穿70, 添加离场信号: exit_long = 1
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], self.sell_rsi.value)) &  # Signal: RSI crosses above sell_rsi
                (dataframe['volume'] > 0)  # Make sure Volume is not 0
            ),
            'exit_long'] = 1
        return dataframe
```

## 策略回测

1. 修改配置

由于默认配置中使用了"pairlists.method=VolumePairList", 即根据货币对在交易所24 小时内交易量筛选可交易货币对, 这种方式不支持回测, 因此我们需要修改.

修改"pairlists.method=StaticPairList", "exchange.pair_whitelist=["BTC/USDT", "ETH/USDT"]", 即仅测试我们设置的白名单中的货币对

![]({{site.baseurl}}/images/202302222117.png)

2. 下载数据

根据当前目录下的配置, 将数据下载到当前目录

```python
freqtrade download-data --userdir ./
```

3. 启动回测

```python
freqtrade backtesting --strategy SampleStrategy --userdir ./
```

![]({{site.baseurl}}/images/202302222121.png)

## 参数优化

寻找最优夏普比率的参数组合

```python
freqtrade hyperopt --strategy SampleStrategy --userdir ./ --hyperopt-loss SharpeHyperOptLossDaily
```

![]({{site.baseurl}}/images/202302222136.png)

根据优化结果将这些结果填写到策略中, 然后重新执行回测命令, 可以看到我们的结果和最优结果一致

![]({{site.baseurl}}/images/202302222139.png)

## 总结

更加标准的做法是将历史数据分为两部分, 前一部分(训练集))用来做回测及参数优化, 后一部分(测试集)用优化的结果再测试. 如果测试集的结果也表现良好, 才能说明我们的策略"及格", 否则只能说明在参数优化过程中, **参数过拟合**了.

我们已经完成了一个策略的基本准备工作, 尝试编写你自己的策略并测试效果吧.

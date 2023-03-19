---
date: 2023-03-09
title: Freqtrade策略_多均线
categories:
  - strategy
  - freqtrade
author_staff_member: dumengru
---

本文讲解使用Freqtrade编写的多均线策略

## freqtrade-strategies

Freqtrade官方提供了一些策略示例代码: https://github.com/freqtrade/freqtrade-strategies

建议将这些文件放在自己的"freqtrade/user_data"目录下

本文使用"MultiMa.py"文件

## 注意事项

#### 配置优先级

命令行中有配置参数, config.json文件中有配置参数, 策略文件中也有配置参数, 当它们冲突时, 优先级由高到低按如下排列

1. 命令行参数(优先级最高)
2. 电脑环境变量(很少用, 但需要注意)
3. 配置文件(config.json)
4. 策略文件(优先级最低)

**注意: 尤其需要注意当config.json和策略中出现相同参数时, 策略文件中的配置会被忽略**

#### 优化参数

Freqtrade提供的策略文件中都会放入已经经过参数优化后筛选的最优参数, 他们会覆盖参数优化中的默认参数. 我粘贴的源码中有注释

## 策略解析

```python
class MultiMa(IStrategy):
    # 策模板版本
    INTERFACE_VERSION: int = 3
    # ***** 以下是经过参数优化后的结果 ***** #
    # 回测时, 这里的buy_params和sell_params会覆盖下面的buy_ma_count等默认参数
    buy_params = {
        "buy_ma_count": 4,
        "buy_ma_gap": 15,
    }
    sell_params = {
        "sell_ma_count": 12,
        "sell_ma_gap": 68,
    }
    minimal_roi = {
        "0": 0.523,
        "1553": 0.123,
        "2332": 0.076,
        "3169": 0
    }
    # 止损
    stoploss = -0.345
    # 关闭跟踪止损
    trailing_stop = False  
    trailing_stop_positive = None
    trailing_stop_positive_offset = 0.0
    trailing_only_offset_is_reached = False
    # ***** 以上是参数优化后的结果 ***** #

    # 时间周期
    timeframe = "4h"

    count_max = 20
    gap_max = 100
    # ***** 以下是可以进行参数优化的参数 ***** # 
    # 整型参数: 从1到count_max, 回测时默认值为7
    buy_ma_count = IntParameter(1, count_max, default=7, space="buy")
    buy_ma_gap = IntParameter(1, gap_max, default=7, space="buy")

    sell_ma_count = IntParameter(1, count_max, default=7, space="sell")
    sell_ma_gap = IntParameter(1, gap_max, default=94, space="sell")
    # ***** 以上是可以进行参数优化的参数 ***** # 

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """ 为了能进行参数优化, 这里需要将所有优化空间的指标都计算好 """

        # 这里直接使用源码会收到警告( Consider joining all columns at once using pd.concat(axis=1) instead...), 我换一种写法
        # dataframe中的原始列名
        raw_columns = dataframe.columns.tolist()
        # 保存计算的指标, 应该是Series类型
        indicators = []
        # 保存对应的指标列名
        indicators_columns = []
        for count in range(self.count_max):
            for gap in range(self.gap_max):
                if count*gap > 1 and count*gap not in indicators_columns:
                    # 将没有重复的指标和指标名按顺序保存到列表中
                    indicators.append(
                        ta.TEMA(
                            dataframe, timeperiod=int(count * gap)
                        )
                    )
                    indicators_columns.append(count*gap)
        # 使用pd.concat横向合并所有指标值, 并添加列名
        dataframe = pd.concat([dataframe] + indicators, axis=1).copy()
        dataframe.columns = raw_columns + indicators_columns

        print(" ", metadata['pair'], end="\t\r")
        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        conditions = []
        # 入场条件判断, 
        # 遍历: 0-self.buy_ma_count.value(假设是7)
        for ma_count in range(self.buy_ma_count.value):
            key = ma_count*self.buy_ma_gap.value
            past_key = (ma_count-1)*self.buy_ma_gap.value
            # 假设key=3*7, past_key=2*7
            if past_key > 1 and key in dataframe.keys() and past_key in dataframe.keys():
                # 入场条件是: 21周期的TEMA均线值 < 14周期的TEMA均线值
                conditions.append(dataframe[key] < dataframe[past_key])
        # reduce函数用法(略), 即: 添加一列"enter_long", 满足上述for循环的所有条件时记为1
        if conditions:
            dataframe.loc[reduce(lambda x, y: x & y, conditions), "enter_long"] = 1
        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """ 参考: populate_entry_trend """
        conditions = []

        for ma_count in range(self.sell_ma_count.value):
            key = ma_count*self.sell_ma_gap.value
            past_key = (ma_count-1)*self.sell_ma_gap.value
            if past_key > 1 and key in dataframe.keys() and past_key in dataframe.keys():
                conditions.append(dataframe[key] > dataframe[past_key])

        if conditions:
            dataframe.loc[reduce(lambda x, y: x | y, conditions), "exit_long"] = 1
        return dataframe
```

该策略用一句话总结: 即一系列固定间隔的长周期TEMA均线值小于短周期TEMA均线值时入场做多, 大于时离场, 应该属于趋势类型策略.

**配置文件**

```json
{
    "max_open_trades": 5,
    "stake_currency": "USDT",
    "stake_amount": 20,
    "tradable_balance_ratio": 0.99,
    "fiat_display_currency": "USD",
    "dry_run": true,
    "dry_run_wallet": 1000,
    "cancel_open_orders_on_exit": false,
      "trading_mode": "futures",
      "margin_mode": "isolated",
    "unfilledtimeout": {
        "entry": 10,
        "exit": 10,
        "exit_timeout_count": 0,
        "unit": "minutes"
    },
    "entry_pricing": {
        "price_side": "same",
        "use_order_book": true,
        "order_book_top": 1,
        "price_last_balance": 0.0,
        "check_depth_of_market": {
            "enabled": false,
            "bids_to_ask_delta": 1
        }
    },
    "exit_pricing":{
        "price_side": "same",
        "use_order_book": true,
        "order_book_top": 1
    },
    "exchange": {
        "name": "binance",
        "key": "",
        "secret": "",
        "ccxt_config": {},
        "ccxt_async_config": {},
        "pair_whitelist": [
            "BTC/USDT:USDT",
            "ETH/USDT:USDT"
        ],
        "pair_blacklist": [
            "BNB/.*"
        ]
    },
    "pairlists": [
        {
            "method": "StaticPairList",
            "number_assets": 20,
            "sort_key": "quoteVolume",
            "min_value": 0,
            "refresh_period": 1800
        }
    ],
    "telegram": {
        "enabled": false,
        "token": "",
        "chat_id": ""
    },
    "api_server": {
        "enabled": false,
        "listen_ip_address": "127.0.0.1",
        "listen_port": 8080,
        "verbosity": "error",
        "enable_openapi": false,
        "jwt_secret_key": "",
        "ws_token": "",
        "CORS_origins": [],
        "username": "",
        "password": ""
    },
    "bot_name": "freqtrade",
    "initial_state": "running",
    "force_entry_enable": false,
    "internals": {
        "process_throttle_secs": 5
    },
    "strategy_path": "user_data/strategies/",
    "db_url": "sqlite:///user_data/dryrun_sqlite/tradesv3.dryrun.Strategy.sqlite",
    "logfile": "user_data/logs/dryrun.Strategy.log"
}
```

## 回测及优化

1. 数据下载命令

```python
freqtrade download-data -t 4h -p BTC/USDT:USDT ETH/USDT:USDT --timerange 20220101-
```

2. 回测命令

```python
freqtrade backtesting --strategy MultiMa --config user_data/config.json
```

3. 回测结果如下

![]({{site.baseurl}}/images/202303032210.png)

## 参数优化

将数据分为两部分: 2022年数据作为训练集, 挑选参数, 2023年数据作为测试集验证参数是否有效

1. 参数优化命令

- 优化目标: SharpeHyperOptLossDaily
- 优化空间: default
- 迭代200次
- 回测时间: 20220101-20230101

```python
freqtrade hyperopt --hyperopt-loss SharpeHyperOptLossDaily --spaces default --strategy MultiMa --config user_data/config.json -e 200 --timerange 20220101-20230101
```

2. 优化结果如下(此时策略文件夹下有一个"PatternRecognition.json"的文件, 内容和优化后的结果参数一模一样, 再次运行回测会使用最优结果)

![]({{site.baseurl}}/images/202303032306.png)

3. 列出全部优化结果

```python
freqtrade hyperopt-list
```

4. 将优化结果导出到csv文件

```python
freqtrade hyperopt-list --export-csv result.csv
```

![]({{site.baseurl}}/images/202303032359.png)

5. 输出编号为134(利润最高)的优化结果

![]({{site.baseurl}}/images/202303040002.png)

6. 策略文件夹下的"MultiMa.json"文件内容和选择的参数结果一模一样. 我们可以直接运行回测命令

![]({{site.baseurl}}/images/202303040005.png)

我们也可以将参数结果复制到策略文件中, 像演示的策略一样

![]({{site.baseurl}}/images/202303040009.png)

7. 重新回测2023年的数据

```python
freqtrade backtesting --strategy MultiMa --config user_data/config.json --timerange 20230101-
```

![]({{site.baseurl}}/images/202303040012.png)

该策略无成交. 其实从之前的回测结果也能看出, 该策略最大的问题就是成交次数太少, 收益最高的前几个参数在2022年一整年的时间只有10笔左右的交易. 因此在测试集(20230101-20230303)没有成交也很正常, 我们暂时无法评估参数的好坏

## 总结

也许我们可以通过增加货币对(本文只写了两个)从而增加总的交易次数, 也可以通过修改交易时间周期(当前时4h)提高交易频率. 最终策略结果如何就需要自己花时间探索了

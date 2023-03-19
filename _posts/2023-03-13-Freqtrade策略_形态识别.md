---
date: 2023-03-13
title: Freqtrade策略_形态识别
categories:
  - strategy
  - freqtrade
author_staff_member: dumengru
---

本文讲解使用Freqtrade编写的形态识别策略

## freqtrade-strategies

Freqtrade官方提供了一些策略示例代码: https://github.com/freqtrade/freqtrade-strategies

建议将这些文件放在自己的"freqtrade/user_data"目录下

本文使用"PatternRecognition.py"文件

## 注意事项

#### 配置优先级

命令行中有配置参数, config.json文件中有配置参数, 策略文件中也有配置参数, 当它们冲突时, 优先级由高到低按如下排列

1. 命令行参数(优先级最高)
2. 电脑环境变量(很少用, 但需要注意)
3. 配置文件(config.json)
4. 策略文件(优先级最低)

**注意: 尤其需要注意当config.json和策略中出现相同参数时, 策略文件中的配置会被忽略**

#### 参数优化

本策略中只有space="buy"空间, 没有sell空间, 因此参数优化选择优化空间时不能选择default, 因为default中包含了sell

## 策略解析

```python
class PatternRecognition(IStrategy):
    # 版本
    INTERFACE_VERSION: int = 3
    # 参数优化后的筛选结果
    buy_params = {
        "buy_pr1": "CDLHIGHWAVE",
        "buy_vol1": -100,
    }

    minimal_roi = {
        "0": 0.936,
        "5271": 0.332,
        "18147": 0.086,
        "48152": 0
    }
    stoploss = -0.288
    # 启用跟踪止损
    trailing_stop = True
    trailing_stop_positive = 0.032
    trailing_stop_positive_offset = 0.084
    trailing_only_offset_is_reached = True

    # 时间周期
    timeframe = '1d'
    # 获取talib中所有的形态函数
    prs = talib.get_function_groups()['Pattern Recognition']

    # 设置可优化参数: buy_pr1形态函数, buy_vol1形态值(0表示没有出现该形态)
    buy_pr1 = CategoricalParameter(prs, default=prs[0], space="buy")
    buy_vol1 = CategoricalParameter([-100,100], default=0, space="buy")

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 计算talib中所有形态指标
        for pr in self.prs:
            dataframe[pr] = getattr(ta, pr)(dataframe)
        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[
            (
                (dataframe[self.buy_pr1.value]==self.buy_vol1.value)
            ),
            'enter_long'] = 1

        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 离场信号(被注释了, 但是策略仍然可通过minimal_roi和止损方式离场)
        dataframe.loc[
            (
            #     (dataframe[self.sell_pr1.value]==self.sell_vol1.value)|
            #     (dataframe[self.sell_pr2.value]==self.sell_vol2.value)
            ),
            'exit_long'] = 1

        return dataframe
```

该策略用一句话总结: 识别K线形态入场并通过风控被动离场

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
        "enabled": true,
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
freqtrade download-data -t 1d -p BTC/USDT:USDT ETH/USDT:USDT --timerange 20220101-
```

2. 回测命令

```python
freqtrade backtesting --strategy PatternRecognition --config user_data/config.json
```

3. 回测结果如下

![]({{site.baseurl}}/images/202303041941.png)

## 参数优化

将数据分为两部分: 2022年数据作为训练集, 挑选参数, 2023年数据作为测试集验证参数是否有效

1. 参数优化命令

- 优化目标: SharpeHyperOptLossDaily
- 优化空间: default
- 迭代100次
- 回测时间: 20220101-20230101

```python
freqtrade hyperopt --hyperopt-loss SharpeHyperOptLossDaily --spaces buy roi stoploss trailing --strategy PatternRecognition --config user_data/config.json -e 100 --timerange 20220101-20230101
```

2. 优化结果如下(此时策略文件夹下有一个"PatternRecognition.json"的文件, 内容和优化后的结果参数一模一样, 再次运行回测会使用最优结果)

![]({{site.baseurl}}/images/202303042004.png)

3. 列出全部优化结果

```python
freqtrade hyperopt-list
```

4. 将优化结果导出到csv文件

```python
freqtrade hyperopt-list --export-csv result.csv
```

![]({{site.baseurl}}/images/202303042009.png)

5. 输出编号为61(利润最高)的优化结果(策略文件夹下的"PatternRecognition.json"的文件, 内容和选择的优化结果参数一模一样, 可以直接运行回测命令)

```python
freqtrade hyperopt-show -n 61
```

6. 该策略的交易信号数量太少, 因此在测试集验证也就没有必要了(策略需要改进)

## 总结

该形态识别策略和之前讲过的多均线策略一样交易信号太少, 但是形态识别策略还有很大改进空间, 我们不仅可以通过降低时间周期(由1d改为1h)增加交易信号, 也可以通过增加做空方式, 增加可交易货币对, 增加形态和指标组合等等改进交易信号

---
date: 2023-03-10
title: Freqtrade策略_多周期
categories:
  - strategy
  - freqtrade
author_staff_member: dumengru
---

本文讲解使用Freqtrade编写的多周期策略

## freqtrade-strategies

Freqtrade官方提供了一些策略示例代码: https://github.com/freqtrade/freqtrade-strategies

建议将这些文件放在自己的"freqtrade/user_data"目录下

本文使用"multi_tf.py"文件

## 注意事项

#### 配置优先级

命令行中有配置参数, config.json文件中有配置参数, 策略文件中也有配置参数, 当它们冲突时, 优先级由高到低按如下排列

1. 命令行参数(优先级最高)
2. 电脑环境变量(很少用, 但需要注意)
3. 配置文件(config.json)
4. 策略文件(优先级最低)

**注意: 尤其需要注意当config.json和策略中出现相同参数时, 策略文件中的配置会被忽略**

#### informative装饰器

在文章"Freqtrade基础_策略编写"中"获取额外数据"一节, 我提到过使用

如果在策略中需要某些货币对的相关数据, 可以添加"informative_pairs"方法, 但在实际中我发现**该方法在回测中无法使用(在线运行可以使用)**.

```python
def informative_pairs(self):
    return [("ETH/USDT", "5m"),
            ("BTC/USDT", "15m"),
            ]
```

还有另外一种使用informative装饰器的方法, 可以达到同样的效果并且可以在回测中使用, 因此推荐该方法

```python
from datetime import datetime
from freqtrade.persistence import Trade
from freqtrade.strategy import IStrategy, informative

class AwesomeStrategy(IStrategy):

    """
    添加当前货币对1h数据, 添加列名:
    'date_1h', 'open_1h', 'high_1h', 'low_1h', 'close_1h', 'volume_1h', 'rsi_1h'
    """
    @informative('1h')
    def populate_indicators_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        return dataframe

    """
    添加以BTC为标的货币以配置中的stake_currency为计价货币的数据, 添加列名:
    'btc_usdt_date_1h', 'btc_usdt_open_1h', 'btc_usdt_high_1h', 'btc_usdt_low_1h',
    'btc_usdt_close_1h', 'btc_usdt_volume_1h', 'btc_usdt_rsi_1h'

    """
    @informative('1h', 'BTC/{stake}:USDT')
    def populate_indicators_btc_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        return dataframe

    """
    自定义列名形式, 添加列名:
    'aa_btc_usdt_date_1h', 'aa_btc_usdt_open_1h', 'aa_btc_usdt_high_1h', 'aa_btc_usdt_low_1h', 'aa_btc_usdt_close_1h', 'aa_btc_usdt_volume_1h', 'aa_btc_usdt_rsi_upper_1h'
    """
    @informative('1h', 'BTC/{stake}:USDT', 'aa_{base}_{quote}_{column}_{timeframe}')
    def populate_indicators_btc_1h_2(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi_upper'] = ta.RSI(dataframe, timeperiod=14)
        return dataframe
```

#### 其他事项

1. 该策略使用的是现货(spot: BTC/USDT), 而不是以往我常用的期货(futures: BTC/USDT:USDT), 注意修改配置和名称区别
2. 该策略使用多周期多品种数据, 回测前一定要先下载好所有需要的货币对数据
3. 该策略中没有添加可优化的参数, 因此无法直接进行参数优化

## 策略解析

```python
class multi_tf (IStrategy):
    # 策略版本
    INTERFACE_VERSION = 3

    # 风控:
    minimal_roi = {
        "0": 0.2
    }
    # 止损
    stoploss = -0.1
    # 跟踪止损(关闭)
    trailing_stop = False
    trailing_stop_positive = 0.001
    trailing_stop_positive_offset = 0.01
    trailing_only_offset_is_reached = True

    # 这几个都是默认参数, 多余
    use_exit_signal = True
    exit_profit_only = False
    exit_profit_offset = 0.01
    ignore_roi_if_entry_signal = False

    # 策略时间周期
    timeframe = '5m'
    # 仅出现新K线计算一次
    process_only_new_candles = True
    # 起始K线数量
    startup_candle_count = 100

    # ***** 以下内容参考上文: informative装饰器 ***** #
    @informative('30m')
    @informative('1h')
    def populate_indicators_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        return dataframe

    @informative('1h', 'BTC/{stake}')
    def populate_indicators_btc_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        return dataframe

    @informative('1h', 'ETH/BTC')
    def populate_indicators_eth_btc_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        return dataframe

    @informative('1h', 'BTC/{stake}', 'BTC_{column}_{timeframe}')
    def populate_indicators_btc_1h_2(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi_fast_upper'] = ta.RSI(dataframe, timeperiod=4)
        return dataframe

    @informative('1h', 'BTC/{stake}', '{base}_{column}_{timeframe}')
    def populate_indicators_btc_1h_3(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi_super_fast'] = ta.RSI(dataframe, timeperiod=2)
        return dataframe

    # ***** 以上内容参考上文: informative装饰器 ***** #

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 计算14周期RSI指标值(策略周期是5m)
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        # 添加一列布尔类型指标: rsi_5m < rsi_1h
        dataframe['rsi_less'] = dataframe['rsi'] < dataframe['rsi_1h']
        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """ 计算入场信号 """
        # stake: usdt
        stake = self.config['stake_currency'].lower()
        # 入场条件判断(指标见名知意, 略)
        dataframe.loc[
            (
                (dataframe[f'btc_{stake}_rsi_1h'] < 35)
                &
                (dataframe['eth_btc_rsi_1h'] < 50)
                &
                (dataframe['BTC_rsi_fast_upper_1h'] < 40)
                &
                (dataframe['btc_rsi_super_fast_1h'] < 30)
                &
                (dataframe['rsi_30m'] < 40)
                &
                (dataframe['rsi_1h'] < 40)
                &
                (dataframe['rsi'] < 30)
                &
                (dataframe['rsi_less'] == True)
                &
                (dataframe['volume'] > 0)
            ),
            ['enter_long', 'enter_tag']] = (1, 'buy_signal_rsi')
            # 给入场信号添加了一列标签: buy_signal_rsi
        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """ 计算离场信号 """
        dataframe.loc[
            (
                (dataframe['rsi'] > 70)
                &
                (dataframe['rsi_less'] == False)
                &
                (dataframe['volume'] > 0)
            ),
            ['exit_long', 'exit_tag']] = (1, 'exit_signal_rsi')

        return dataframe

```

该策略用一句话总结: 根据不同周期和相关货币对的RSI指标值作为信号, 根据信号入场和离场. (该策略除了堆叠指标值, 似乎毫无逻辑)

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
      "trading_mode": "spot",
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
            "BTC/USDT",
            "ETH/USDT"
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
freqtrade download-data -t 5m 1h 30m -p BTC/USDT ETH/USDT ETH/BTC --timerange 20220101-
```

2. 回测命令

```python
freqtrade backtesting --strategy multi_tf --config user_data/config.json
```

3. 回测结果如下

![]({{site.baseurl}}/images/202303041250.png)

## 总结

该策略主要目的是为了学习多周期和多品种策略编写方法, 缺少可优化的参数空间. 感兴趣的可以自己尝试添加参数优化相关内容, 需要注意的是如果要优化指标参数相关内容, 一定要在"populate_indicators"方法中把所有指标数据都计算出来

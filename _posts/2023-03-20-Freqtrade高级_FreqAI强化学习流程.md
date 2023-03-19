---
date: 2023-03-20
title: Freqtrade高级_FreqAI强化学习流程
categories:
  - freqtrade
  - ai
author_staff_member: dumengru
---

本文梳理了FreqAI中强化学习模型的工作流程

![]({{site.baseurl}}/images/202302281701.png)

AI相关参考文档:

- scikit-learn文档: https://scikit-learn.org/stable/user_guide.html
- Keras文档: https://keras.io/examples/
- PyTorch文档: https://pytorch.org/tutorials/

- 动手学深度学习: http://zh.d2l.ai/
- 动手学强化学习: https://hrl.boyuai.com/chapter/intro

- stable-baselines3: https://stable-baselines3.readthedocs.io/en/master/index.html
- OpenAI gym: https://www.gymlibrary.dev/content/basic_usage/

## 准备事项

1. 新建"user_data/strategies/SampleStrategy.py"文件

```python
class SampleStrategy(IStrategy):
    """
    # 智能体动作: 中性(0), 多头入场(1), 多头离场(2), 空头入场(3), 空头离场(4)
    """
    minimal_roi = {"0": 0.1, "240": -1}

    process_only_new_candles = True
    stoploss = -0.05
    startup_candle_count: int = 40
    can_short = True

    def feature_engineering_expand_all(self, dataframe: DataFrame, period: int, metadata: Dict, **kwargs):
        """ 构造特征 """
        dataframe["%-rsi-period"] = ta.RSI(dataframe, timeperiod=period)
        dataframe["%-mfi-period"] = ta.MFI(dataframe, timeperiod=period)
        dataframe["%-adx-period"] = ta.ADX(dataframe, timeperiod=period)
        dataframe["%-sma-period"] = ta.SMA(dataframe, timeperiod=period)
        dataframe["%-ema-period"] = ta.EMA(dataframe, timeperiod=period)

        bollinger = qtpylib.bollinger_bands(
            qtpylib.typical_price(dataframe), window=period, stds=2.2
        )
        dataframe["bb_lowerband-period"] = bollinger["lower"]
        dataframe["bb_middleband-period"] = bollinger["mid"]
        dataframe["bb_upperband-period"] = bollinger["upper"]

        dataframe["%-bb_width-period"] = (
            dataframe["bb_upperband-period"]
            - dataframe["bb_lowerband-period"]
        ) / dataframe["bb_middleband-period"]
        dataframe["%-close-bb_lower-period"] = (
            dataframe["close"] / dataframe["bb_lowerband-period"]
        )

        dataframe["%-roc-period"] = ta.ROC(dataframe, timeperiod=period)
        dataframe["%-relative_volume-period"] = (
            dataframe["volume"] / dataframe["volume"].rolling(period).mean()
        )

        return dataframe

    def feature_engineering_expand_basic(self, dataframe: DataFrame, metadata: Dict, **kwargs):
        """ 构造特征 """
        dataframe["%-pct-change"] = dataframe["close"].pct_change()
        dataframe["%-raw_volume"] = dataframe["volume"]
        dataframe["%-raw_price"] = dataframe["close"]
        return dataframe

    def feature_engineering_standard(self, dataframe: DataFrame, metadata: Dict, **kwargs):
        """ 构造特征 """
        dataframe["%-day_of_week"] = dataframe["date"].dt.dayofweek
        dataframe["%-hour_of_day"] = dataframe["date"].dt.hour

        # 需要将原始价格数据传入, 以"%-raw"开始
        dataframe[f"%-raw_close"] = dataframe["close"]
        dataframe[f"%-raw_open"] = dataframe["open"]
        dataframe[f"%-raw_high"] = dataframe["high"]
        dataframe[f"%-raw_low"] = dataframe["low"]

        return dataframe

    def set_freqai_targets(self, dataframe: DataFrame, metadata: Dict, **kwargs):
        """ 将该列设为中性值 """
        dataframe["&-action"] = 0
        return dataframe

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """  """
        # 初始化训练过程
        dataframe = self.freqai.start(dataframe, metadata, self)
        return dataframe

    def populate_entry_trend(self, df: DataFrame, metadata: dict) -> DataFrame:
        """ 入场 """
        # 智能体执行多头入场动作, do_predict表示预测数据经过检测是可信的, action=1表示多头入场
        enter_long_conditions = [df["do_predict"] == 1, df["&-action"] == 1]
        if enter_long_conditions:
            df.loc[
                reduce(lambda x, y: x & y, enter_long_conditions), ["enter_long", "enter_tag"]
            ] = (1, "long")
        # 智能体执行多头入场动作, do_predict表示预测数据经过检测是可信的, action=3表示空头入场
        enter_short_conditions = [df["do_predict"] == 1, df["&-action"] == 3]
        if enter_short_conditions:
            df.loc[
                reduce(lambda x, y: x & y, enter_short_conditions), ["enter_short", "enter_tag"]
            ] = (1, "short")
        return df

    def populate_exit_trend(self, df: DataFrame, metadata: dict) -> DataFrame:
        """ 离场 """
        # 智能体执行多头入场动作, do_predict表示预测数据经过检测是可信的, action=2表示多头离场
        exit_long_conditions = [df["do_predict"] == 1, df["&-action"] == 2]
        if exit_long_conditions:
            df.loc[reduce(lambda x, y: x & y, exit_long_conditions), "exit_long"] = 1
        # 智能体执行多头入场动作, do_predict表示预测数据经过检测是可信的, action=4表示空头离场
        exit_short_conditions = [df["do_predict"] == 1, df["&-action"] == 4]
        if exit_short_conditions:
            df.loc[reduce(lambda x, y: x & y, exit_short_conditions), "exit_short"] = 1
        return df
```

2. 使用的配置文件在"config_examples/config_freqai.example.json"文件基础上做了修改

```python
{
    "trading_mode": "futures",
    "margin_mode": "isolated",
    "max_open_trades": 5,
    "stake_currency": "USDT",
    "stake_amount": 200,
    "tradable_balance_ratio": 1,
    "fiat_display_currency": "USD",
    "dry_run": true,
    "timeframe": "5m",
    "dry_run_wallet": 1000,
    "cancel_open_orders_on_exit": true,
    "unfilledtimeout": {
        "entry": 10,
        "exit": 30
    },
    "exchange": {
        "name": "binance",
        "key": "",
        "secret": "",
        "ccxt_config": {},
        "ccxt_async_config": {},
        "pair_whitelist": [
            "ETH/USDT:USDT"
        ],
        "pair_blacklist": []
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
    "exit_pricing": {
        "price_side": "other",
        "use_order_book": true,
        "order_book_top": 1
    },
    "pairlists": [
        {
            "method": "StaticPairList"
        }
    ],
    "freqai": {
        "rl_config": {
            "train_cycles": 25,
            "add_state_info": true,
            "max_trade_duration_candles": 300,
            "max_training_drawdown_pct": 0.02,
            "cpu_count": 8,
            "model_type": "PPO",
            "policy_type": "MlpPolicy",
            "model_reward_parameters": {
                "rr": 1,
                "profit_aim": 0.025
            }
        },
        "enabled": true,
        "purge_old_models": true,
        "train_period_days": 15,
        "backtest_period_days": 7,
        "live_retrain_hours": 0,
        "identifier": "uniqe-id",
        "feature_parameters": {
            "include_timeframes": [
                "1h"
            ],
            "include_corr_pairlist": [
                "BTC/USDT:USDT"
            ],
            "label_period_candles": 20,
            "include_shifted_candles": 2,
            "DI_threshold": 0.9,
            "weight_factor": 0.9,
            "principal_component_analysis": false,
            "use_SVM_to_remove_outliers": true,
            "indicator_periods_candles": [
                10,
                20
            ],
            "plot_feature_importances": 0
        },
        "data_split_parameters": {
            "test_size": 0.33,
            "random_state": 1
        },
        "model_training_parameters": {}
    },
    "bot_name": "",
    "force_entry_enable": true,
    "initial_state": "running",
    "internals": {
        "process_throttle_secs": 5
    }
}
```

3. 回测命令

```python
freqtrade backtesting --strategy SampleStrategy --strategy-path user_data/strategies --config config_examples/config_freqai.example.json --freqaimodel ReinforcementLearner --timerange 20220501-20230101
```

4. 上述配置和命令解释
- 交易货币对: ETH/USDT:USDT
- 交易的时间周期: 5m
- 添加特征货币对: BTC/USDT:USDT
- 添加特征时间周期: 1h
- 训练数据(训练集和测试集)15天一批次: train_period_days
- 测试集占比: 0.33
- 验证集数据7天一批次: backtest_period_days
- 回测时间20220501-20230101
- 使用的强化学习模型: ReinforcementLearner

## 回测流程

1. 首先进入策略的"populate_indicators"方法, 执行

```python
dataframe = self.freqai.start(dataframe, metadata, self)
```

所有的数据处理, 模型训练和预测等都通过上述命令执行完

2. 进入"freqai_interface.py"文件中的"start"方法, 执行

```python
dk = self.start_backtesting(dataframe, metadata, self.dk, strategy)
```

所有的数据处理, 模型训练和预测等都通过上述命令执行完

3. 进入"freqai_interface.py"文件中的"start_backtesting"方法, 所有回测流程逻辑都在此

模型批次训练逻辑都在for循环中

```python
for tr_train, tr_backtest in zip(dk.training_timeranges, dk.backtesting_timeranges)
```

- dk.training_timeranges: 保存训练数据时间区间列表, 第一批次起始时间会从20220501向前推15天, 第一批次的结束时间就是20220501; 第二批次从20220501向后推15天, 以此类推
- dk.backtesting_timeranges: 保存验证集数据时间区间列表, 第一批次从20220501向后推7天, 以此类推
- tr_train: 即每个批次训练数据的时间区间
- tr_backtest: 即每个批次验证集数据时间区间

4. for循环内部
- self.dk.use_strategy_to_populate_indicators: 计算策略中所编写的所有特征数据, 由于多批次训练, 为避免重复计算, 通过"populate_indicators = False"控制
- dataframe_train: 获取当前批次的训练数据
- dataframe_backtest: 获取当前批次验证集数据
- dk.find_features: 获取所有特征列名, 以"%"开头的列都是特征列
- dk.find_labels: 获取所有标签列名, 以"&"开头的列都是标签列
- self.train: 模型训练过程, 这里会跳转到子类"freqtrade/freqai/RL/BaseReinforcementLearningModel.py/BaseReinforcementLearningModel"
- plot_feature_importance: 绘制特征重要性图
- self.dd.save_data: 保存模型
- self.predict: 模型预测过程, 这里会跳转到子类"freqtrade/freqai/RL/BaseReinforcementLearningModel.py/BaseReinforcementLearningModel"
- 将验证集标签, 预测值和DI值合并: dk.get_predictions_to_append
- 将训练数据和验证数据合并: dk.append_predictions
- 保存预测结果到本地: dk.save_backtesting_prediction

for循环结束, 还有两步

- 利用回测数据, 预测未来标签: backtesting_fit_live_predictions
- 合并所有的价格, 特征, 标签和预测数据: fill_predictions

至此FreqAI回测逻辑基本结束, 我们已经根据训练出模型并获得了所有预测值, 剩下的内容将预测值看作指标值, 再根据该值做出买卖信号和传统回测逻辑一模一样

## 模型训练和预测

回测流程中最重要的两步, 模型训练和预测都写在"freqtrade/freqai/RL/BaseReinforcementLearningModel.py/BaseReinforcementLearningModel"中

#### train

模型训练步骤

1. 获取当前训练批次数据中的特征列和标签列数据: dk.filter_features 
2. 利用sklearn中的`train_test_split`方法构造训练集和测试集数据: dk.make_train_test_datasets
3. 利用scipy计算训练集中的均值和标准差: dk.fit_labels
4. 将以"%-raw_"开头的原始价格数据添加到数据集中
5. 通过训练集数据将训练集和测试集数据归一化: dk.normalize_data
6. 最后数据处理(删除异常值添加噪音等): self.data_cleaning_train
7. 创建模型训练环境: self.set_train_and_eval_environments
7. 模型训练: self.fit

#### fit细节

训练模型具体步骤及奖励函数逻辑在"freqtrade/freqai/prediction_models/ReinforcementLearner.py/ReinforcementLearner"中

1. 获取训练集数据特征: train_df
2. 计算迭代总次数: total_timesteps = 配置`train_cycles`*len(train_df)
3. 获取神经网络参数: policy_kwargs
4. 如果配置`continual_learning=False`, 重新构造SB3模型, 否则使用原有模型
5. 设置模型环境: model.set_env
6. 模型学习: model.learn
7. 如果发现最优模型模型立即返回

#### predict

模型预测步骤

1. 获取特征标签: dk.find_features
2. 获取验证集特征列: dk.filter_features
3. 根据训练集数据归一化验证集数据:  dk.normalize_data_from_metadata
4. 最后的数据处理: self.data_cleaning_predict
5. 模型预测: self.rl_model_predict
6. 预测数据填充空值: pred_df.fillna

## 最后的数据处理

在模型训练和预测过程中最后的数据处理逻辑在"freqtrade/freqai/freqai_interface.py/IFreqaiModel.data_cleaning_train"

#### data_cleaning_train

最后的数据处理, 这些都可以通过参数配置

- inlier_metric_window: 衡量当前数据点与最近的历史数据点的相似程度
- principal_component_analysis: PCA降维
- use_SVM_to_remove_outliers: SVM移除异常值
- DI_threshold: 使用DI过滤异常值
- use_DBSCAN_to_remove_outliers: 使用DBSCAN聚类识别异常值
- noise_standard_deviation: 向训练特征中添加噪声防止数据过拟合

## 回测结果

回测太耗时, 因此我仅用了20221201-20230101这段数据, 结果如下

![]({{site.baseurl}}/images/202303082148.png)

## 流程总结

以上便是FreqAI强化学习的全部工作流程, 其大体工作流程和机器学习一模一样, 区别在于强化学习不需要标签, 但需要合适的奖励函数

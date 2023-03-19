---
date: 2023-03-16
title: Freqtrade高级_FreqAI流程详述
categories:
  - freqtrade
  - ai
author_staff_member: dumengru
---

本文梳理了FreqAI中机器学习模型的工作流程

![]({{site.baseurl}}/images/202302281701.png)

AI相关参考文档:

- scikit-learn文档: https://scikit-learn.org/stable/user_guide.html
- Keras文档: https://keras.io/examples/
- PyTorch文档: https://pytorch.org/tutorials/

- 动手学深度学习: http://zh.d2l.ai/
- 动手学强化学习: https://hrl.boyuai.com/chapter/intro

- stable-baselines3: https://stable-baselines3.readthedocs.io/en/master/index.html
- OpenAI gym: https://www.gymlibrary.dev/content/basic_usage/

## 简介

利用机器学习进行量化交易最复杂的地方就是数据处理部分, 即特征工程. 由于机器学习可以非常容易拟合交易历史数据, 如果我们在数据处理过程中有错误或不合理的地方, 很容易在回测数据中添加未来数据或是造成过拟合. 因此有必要详细梳理下FreqAI的数据处理流程了

## 准备事项

1. 使用的策略是"freqtrade/templates/FreqaiExampleStrategy.py"
2. 使用的配置文件在"config_examples/config_freqai.example.json"文件基础上做了简化

```json
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
freqtrade backtesting --strategy FreqaiExampleStrategy --strategy-path freqtrade/templates --config config_examples/config_freqai.example.json --freqaimodel LightGBMRegressor --timerange 20220501-20230101
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
- 使用的机器学习模型: LightGBMRegressor

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
- self.train: 模型训练过程, 这里会跳转到子类"freqtrade/freqai/base_models/BaseRegressionModel.py/BaseRegressionModel"
- plot_feature_importance: 绘制特征重要性图
- self.dd.save_data: 保存模型
- self.predict: 模型预测过程, 这里会跳转到子类"freqtrade/freqai/base_models/BaseRegressionModel.py/BaseRegressionModel"
- 将验证集标签, 预测值和DI值合并: dk.get_predictions_to_append
- 将训练数据和验证数据合并: dk.append_predictions
- 保存预测结果到本地: dk.save_backtesting_prediction

for循环结束, 还有两步

- 利用回测数据, 预测未来标签: backtesting_fit_live_predictions
- 合并所有的价格, 特征, 标签和预测数据: fill_predictions

至此FreqAI回测逻辑基本结束, 我们已经根据训练出模型并获得了所有预测值, 剩下的内容将预测值看作指标值, 再根据该值做出买卖信号和传统回测逻辑一模一样

## 模型训练和预测

回测流程中最重要的两步, 模型训练和预测都写在"freqtrade/freqai/base_models/BaseRegressionModel.py/BaseRegressionModel"中

#### train

模型训练步骤

1. 获取当前训练批次数据中的特征列和标签列数据: dk.filter_features 
2. 获取当前训练批次数据的起始日期和结束日期: start_date, end_date
3. 利用sklearn中的`train_test_split`方法构造训练集和测试集数据: dk.make_train_test_datasets
4. 利用scipy计算训练集中的均值和标准差: dk.fit_labels
5. 通过训练集数据将训练集和测试集数据归一化: dk.normalize_data
6. 最后数据处理(删除异常值添加噪音等): self.data_cleaning_train
7. 模型训练: self.fit

#### predict

模型预测步骤

1. 获取特征标签: dk.find_features
2. 获取验证集特征列: dk.filter_features
3. 根据训练集数据归一化验证集数据:  dk.normalize_data_from_metadata
4. 最后的数据处理: self.data_cleaning_predict
5. 模型预测: self.model.predict
6. 反归一预测数据: dk.denormalize_labels_from_metadata

#### fit细节

训练模型具体步骤在"freqtrade/freqai/prediction_models/LightGBMRegressor.py/LightGBMRegressor"中

1. 获取训练集数据和权重: eval_set, eval_weights
2. 获取训练特征, 标签和权重因子: x, y, train_weights
3. 查看是否有初始模型模型: self.get_init_model
4. 根据配置"model_training_parameters"构造LGBMRegressor模型
5. 模型训练: model.fit

#### 最后的数据处理

在模型训练和预测过程中最后的数据处理逻辑在"freqtrade/freqai/freqai_interface.py/IFreqaiModel.data_cleaning_train"

**data_cleaning_train**

最后的数据处理, 这些都可以通过参数配置

- inlier_metric_window: 衡量当前数据点与最近的历史数据点的相似程度
- principal_component_analysis: PCA降维
- use_SVM_to_remove_outliers: SVM移除异常值
- DI_threshold: 使用DI过滤异常值
- use_DBSCAN_to_remove_outliers: 使用DBSCAN聚类识别异常值
- noise_standard_deviation: 向训练特征中添加噪声防止数据过拟合

## 流程总结

以上便是FreqAI机器学习的全部工作流程, 基本就是: 构造特征->数据划分->计算批次标签->批次训练->批次预测->数据合并->输出结果. 

其中训练和预测过程: 获取特征->数据归一化和标准化->过滤异常值->添加噪音->训练/预测

![]({{site.baseurl}}/images/202303042232.png)

## 总结

如果你对机器学习不了解, 切勿投入真金白银. 永远不要做自己不懂的事, 尤其是在金融领域

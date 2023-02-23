---
date: 2023-02-24
title: Freqtrade基础_FreqUI和Telegram
categories:
  - knowledge
  - freqtrade
  - tool
author_staff_member: dumengru
---

编写完策略及参数优化后, 我们可以进入仿真测试环节了

## FreqUI

Freqtrade提供了一个UI界面叫FreqUI, 需要单独安装和配置

1. 安装FreqUI

```python
freqtrade install-ui
```

2. 在"config.json"文件中修改配置

```json
"api_server": {
    "enabled": true,
    "listen_ip_address": "127.0.0.1",
    "listen_port": 8080,
    "verbosity": "error",
    "enable_openapi": false,
    "jwt_secret_key": "somethingrandom",
    "CORS_origins": [],
    "username": "freqtrade",
    "password": "freqtrade",
    "ws_token": "sercet_Ws_t0ken"
},
```

3. 运行策略

```python
freqtrade trade --userdir ./ --strategy SampleStrategy
```

4. 打开本地浏览器输入"127.0.0.1:8080"即可看到FreqUI界面

5. 点击右上角登录, 并输入配置的用户名和密码即可查看策略运行状态

![]({{site.baseurl}}/images/202302211413.png)

6. 如果自己有服务器, 将"listen_ip_address"改为自己的服务器地址, 即可在任意网络设备查看策略运行状态(以下是我在手机上打开服务器地址的登陆界面)

![]({{site.baseurl}}/images/202302230903.png)

## Telegram(非必要)

如果在国内无法使用Telegram也不必强求, 详细的配置内容参考官方文档: https://www.freqtrade.io/en/stable/telegram-usage/

![]({{site.baseurl}}/images/202302231003.png)

1. 申请一个机器人, 机器人名称需要以"_bot"结尾

2. 申请用户的机器人id, 如果Telegram新用户没有设置Username, 需要先设置Username

3. 将申请的机器人"token"和机器人id分别填写到"config.json"中的对应位置, 设置"enabled=true"

```json
"telegram": {
    "enabled": true,
    "token": "your_telegram_token",
    "chat_id": "your_telegram_chat_id",
    "allow_custom_messages": true,
    "notification_settings": {
        "status": "silent",
        "warning": "on",
        "startup": "off",
        "entry": "silent",
        "entry_fill": "on",
        "entry_cancel": "silent",
        "exit": {
            "roi": "silent",
            "emergency_exit": "on",
            "force_exit": "on",
            "exit_signal": "silent",
            "trailing_stop_loss": "on",
            "stop_loss": "on",
            "stoploss_on_exchange": "on",
            "custom_exit": "silent",
            "partial_exit": "on"
        },
        "exit_cancel": "on",
        "exit_fill": "off",
        "protection_trigger": "off",
        "protection_trigger_global": "on",
        "strategy_msg": "off",
        "show_candle": "off"
    },
    "reload": true,
    "balance_dust_level": 0.01
},
```

4. 运行策略

```python
freqtrade trade --userdir ./ --strategy SampleStrategy
```

5. 查看自己的机器人对话框, 每当策略执行订单时都会收到消息. 右下角还有一些快捷菜单可以控制机器人

![]({{site.baseurl}}/images/202302231009.png)

## 总结

策略编写中用到了未来函数, 手续费和滑点设置不合理, 参数优化过程中参数过拟合等许多因素都可能导致回测结果失真, 因此即使回测效果足够好, 也不足以说明我们的策略具备实盘条件, **仿真测试是实盘前必须经历的步骤**

值得高兴的是, Freqtrade只需要在配置文件中设置"dry_run=true"(这也是默认设置), 即可进行仿真测试

此外, Freqtrade在运行过程中可能由于各种原因"卡住", 记得不定期检查程序执行情况哟

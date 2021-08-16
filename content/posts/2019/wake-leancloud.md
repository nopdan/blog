---
title: 使用腾讯云函数自动唤醒LeanCloud
categories: 
  - 笔记
tags:
  - leancloud
  - 腾讯云
  - valine
date: 2020-08-10
---

前段时间，leancloud 官方对定时任务添加了流控，导致 valine 自动唤醒失败。  
日志：`"因流控原因，通过定时任务唤醒体验版实例失败，建议升级至标准版云引擎实例避免休眠"。`

<!-- more -->

## 问题由来

![](https://p.pstatp.com/origin/pgc-image/e9612c19799344889a17d72ba7440a62)

## 云引擎体验实例休眠策略

体验实例会执行休眠策略：

- 如果应用最近一段时间（半小时）没有任何外部请求，则休眠。
- 休眠后如果有新的外部请求实例则马上启动。访问者的体验是第一个请求响应时间是 5 ~ 30 秒（视实例启动时间而定），后续访问响应速度恢复正常。
- 强制休眠：如果最近 24 小时内累计运行超过 18 小时，则强制休眠。此时新的请求会收到 503 的错误响应码，该错误可在 云引擎 > 统计 中查看。

> 解决思路：我们只要每三十分钟之内在外部访问一次即可解决。

## 解决方案

使用腾讯云函数从外部定时访问

点击进入 [腾讯云开发控制台](https://console.cloud.tencent.com/tcb)  
新建一个环境，进入后在左侧找到云函数  
点击新建云函数，如图配置  
![](https://p.pstatp.com/origin/pgc-image/b0974e0381c04f1daaa518de08e45dc2)

在启动函数中填入如下代码  
![](https://p.pstatp.com/origin/pgc-image/e4049f6268634da18f07a4387fd56598)

```python
import requests
import time

def main(event, context):
    urls = ["https://example.avosapps.us"] # 此处填入云引擎域名，可以有多项
    for url in urls:
        req = requests.get(url)
        print(f'网址 {url} 唤醒状态: \n', req, time.strftime(
            '%Y-%m-%d %H:%M:%S', time.localtime(time.time())))
```

点击函数名称，在点击右上角的编辑  
在定时触发器处填入如下代码  
![](https://p.pstatp.com/origin/pgc-image/d379eb5d500d40f999285e1787fb9f94)

```json
{
    "triggers": [
        {
            "name": "wake_leancloud",
            "type": "timer",
            "config": "0 1/28 7/1 * * * *"
        }
    ]
}
```

config 项为 cron 表达式，此处表示从 7:01 分开始，每 28 分钟执行一次  
> 由 leancloud 策略，凌晨 0:25 到 7:01 时间段内处于休眠  
> 请将 leancloud 定时任务里 resend_email 的时间调整为 `0 5 23 * * ?`  
> 表示每天 7:05 补漏发邮件 ( leancloud 的 cron 表达式时区为 UTC+0 )

## 参考

[解决 LeanCLoud 定时唤醒失败的流控问题](https://duter2016.github.io/2020/06/09/%E8%A7%A3%E5%86%B3LeanCLoud%E5%AE%9A%E6%97%B6%E5%94%A4%E9%86%92%E5%A4%B1%E8%B4%A5%E7%9A%84%E6%B5%81%E6%8E%A7%E9%97%AE%E9%A2%98/)

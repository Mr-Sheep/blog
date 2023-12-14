---
slug: "rpi-fan-control"
title: "Raspberry Pi Fan Speed"
date: 2021-02-03T00:54:47+08:00
draft: false
tags: ["Raspberry Pi"]
categories: ["fun"]
---

如何使用樹莓派上 GPIO 電壓控制淘寶劣質小風扇速度

<!--more-->

## 水篇文章

很久之前購入的樹莓派小風扇，只能夠接入 power 和 GND，而且接頭還是預拼接在一起的
接入樹莓派 GPIO 4，6 以後轉速直接上天，但是沒辦法拆分到 GPIO 1 和 6 上
所以想着能不能利用 GPIO 13 和 14 搞定

![GPIO-Pinout-Diagram-2.png](https://i.loli.net/2021/02/03/zQVdosZWNe32tXY.png)

不廢話了

```python
import RPi.GPIO as GPIO
GPIO.setmode(GPIO.BOARD)
GPIO.setwarnings(False)
GPIO.setup(13,GPIO.OUT)
state = GPIO.input(13)
if state:
    GPIO.output(13,GPIO.LOW)
    print('Running on 0V')
else:
    GPIO.output(13,GPIO.HIGH)
    print('Running on 3V')

```

重複執行即可達到設定某 GPIO 上的電壓

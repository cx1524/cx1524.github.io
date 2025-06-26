---
title: openocd烧录新方法
published: 2025-06-09
description: "记录一些openocd的基础操作."
image: ""
tags: ["嵌入式", "Openocd"]
category: 嵌入式
draft: true
lang: ''
---

有些时候会出现没有Flash的烧录流程，但有Flash的下载算法文件FLM的情况下去烧录程序，这篇文章将会介绍在这种情景下要如何使用
openocd实现程序烧录

## 原理

参考JLink的烧录流程，jLink会先将FLM中有关烧录的内容加载到SRAM，再将要下载的文件切成n份并加载到指定的SRAM的数据区域中，
再调用SRAM中烧录算法API将SRAM数据区域的数据烧录到Flash中，以此实现烧录。

借助openocd的脚本指令和API指令，我们也可以复刻这一过程。

## 准备

在这套流程中我们将会用到arm-none-eabi工具链，先从[arm-none-eabi-gcc](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads/14-2-rel1)处下载我们需要用到的工具链并在本地解压（可以将其bin/添加进环境变量中方便使用）。

FLM中除了烧录API，还混杂了一些设备信息，如果直接写到SRAM中，对SRAM来说太大了，所以我们需要先将烧录API从FLM中提取出来，
这时就需要用到arm-none-eabi-objcopy对FLM进行提取

```shell
arm-none-eabi-objcopy -O binary input.FLM output.bin
```



使用arm-none-eabi工具链中的arm-none-eabi-objcopy工具，将FLM中的烧录API提取出来，并生成一个.bin文件。

之后我们需要获取烧录API的入口地址，有两种办法可以获取：

第一种方法是使用

从[DAPLink仓库](https://github.com/ARMmbed/DAPLink)克隆源码到本地，再进入`DAPLink/tools`目录，找到一个叫
generate_flash_algo.py的python文件，这就是我们所需要的工具。

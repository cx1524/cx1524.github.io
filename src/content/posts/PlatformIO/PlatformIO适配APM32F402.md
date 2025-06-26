---
title: PlatformIO适配APM32F402
published: 2025-06-26
description: '介绍如何将APM32F402移植进PlatformIO'
image: ''
tags: [嵌入式, PlatformIO]
category: '嵌入式'
draft: false
lang: ''
---

PlatformIO是一个开源的、跨平台的嵌入式开发生态系统，它集成了多种免费开源的嵌入式工具链，并支持多种架构。目前官方也支持了很多主流芯片供开发人员开发。

但官方的支持列表中还没有APM32F402，我们要如何使用PlatformIO平台去开发APM32F402呢？

欸↗↘，因为市面上的芯片百花齐放，官方很难统统适配它们，于是PlatformIO官方直接提供了新芯片的移植方案，方便各路开发者在PlatformIO上适配自己的芯片（适配资料请见PlatformIO官方文档 [Custom Development Platforms篇](https://docs.platformio.org/en/latest/platforms/creating_platform.html)）。

根据PlatformIO的适配文档，接下来我们就一步步地尝试如何适配APM32F402：

## 准备

PlatformIO推荐的开发环境是VSCode + PlatformIO插件，那么我们就准备这套环境：

1. 下载并安装[VSCode](https://code.visualstudio.com/)
2. 从VSCode的插件市场中搜索并安装PlatformIO

[vscode安装PlatformIO](./PlatformIO适配芯片/vscode安装PlatformIO.png)

3. 等待PlatformIO插件将依赖安装完成（因为网络原因，这一步可能会比较慢，请耐心等待）

## 适配

OK，准备完成，接下是适配环节

根据手册上来说，我们需要为我们的芯片准备一个manifest文件（platform.json）和构建脚本（main.py）

### 安装

找到core目录下的platforms文件夹（如果没有则创建一个）

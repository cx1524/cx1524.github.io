---
title: openocd源码打包方法
published: 2025-06-30
description: ''
image: ''
tags: [嵌入式, openocd]
category: '嵌入式'
draft: false
lang: ''
---

有时我们会因为种种需求要去改动openocd的源码来适配我们的设备，那么我们就需要去搭建自己的openocd开发环境，以及打包修改后的openocd定制版来给到别人。这篇文章主要介绍的是openocd的开发环境搭建，以及使用源码打包的方法。

openocd的编译环境为linux，这里以WSL下的Ubuntu系统为例，演示如何完成openocd的开发环境搭建以及windows版本的openocd打包方法。

## 依赖与工具

首先我们需要安装一些openocd打包时要用到的依赖和工具（有一些依赖linux会自带）：

- git
- make
- libtool
- pkg-config
- libjim-dev
- autoconf
- automake
- texinfo
- minge-w64
- build-essential
- libusb-1.0-0-dev (libftdi库要用到)
- cmake (libftdi库要用到)
- libconfuse-dev (libftdi库要用到)

使用指令安装依赖：

```shell
sudo apt update
sudo apt install git make libtool pkg-config libjim-dev autoconf automake texinfo mingw-w64 build-essential libconfuse-dev cmake libusb-1.0-0-dev
```

## 库

接下来我们需要准备一些openocd会用到的库：

- libusb-1.0 （必要）
- libftdi （可选，用于FTDI接口）
- HIDAPI （可选，用于CMSIS-DAP接口）
- libjaylink （可选，用于J-Link接口）

因为我们使用的是CMSIS-DAP，且要打包的是windows版本的openocd，所以我们只需要编译出libusb-1.0和HIDAPI的dll库。

首先用git将这些项目的源码拉取下来：

```shell
git clone https://github.com/libusb/libusb.git
git clone https://github.com/libusb/hidapi.git
```

接着逐个编译它们

- **libusb**

```shell
./bootstrap.sh
mkdir build
cd build
../configure --prefix=${HOME}/Build/libusb --host=x86_64-w64-mingw32
make -j4
make install
```

如何中途出现了错误可以使用`make clean`清除编译文件，重新编译

- **hidapi**

```shell
./bootstrap
mkdir build
cd build
../configure --prefix=${HOME}/Build/hidapi --host=x86_64-w64-mingw32
make -j4
make install
```

此时查看`${HOME}/Build`目录，会发现多了libusb和hidapi两个目录，这两个目录中的bin目录中存放了libusb和hidapi的dll库。

## openocd

接下来开始准备编译openocd，先用git拉取openocd源码：

```shell
git clone git://git.code.sf.net/p/openocd/code openocd
```

> [!TIP]
> 如何使用的第三方openocd，请更改git clone地址。

openocd源码中使用了子模块jimtcl，我们还需更新它的子模块：

```shell
cd openocd
git submodule update --init
```

> [!NOTE]
> 请注意，如果在编译时出现`check for jim.h.. no`错误，则需要我们手动编译
> 一次jimtcl，cd 到jimtcl目录下，执行`./configure`，然后`make`。

接着编译openocd：

```shell
./bootstrap
./configure --prefix=${HOME}/Build/openocd --host=x86_64-w64-mingw32
make -j4
make install
```

此时可以在`${HOME}/Build`目录看到已编译好的openocd文件。

但要想其能运行，我们还需将之前的两个dll库拷贝到openocd的bin目录下，使用命令：

```shell
cp ${HOME}/Build/libusb/bin/libusb-1.0.dll ${HOME}/Build/openocd/bin
cp ${HOME}/Build/hidapi/bin/libhidapi-0.dll ${HOME}/Build/openocd/bin
```

接着将openocd打包成压缩包即可

- tar.gz

```shell
tar -vcf openocd.tar ${HOME}/Build/openocd
gzip openocd.tar
```

- zip

```shell
zip -r openocd.zip ${HOME}/Build/openocd
```

---
title: openocd适配新设备
published: 2025-04-01
description: '记录openocd适配新设备的一些知识'
image: ''
tags: ["嵌入式", "Openocd"]
category: '嵌入式'
draft: true
lang: ''
---

本文章记录的是我在研究openocd的新设备适配流程时所发现的一些规律。

## 适配流程

向openocd中添加新设备，就相当与向openocd中添加一个新的`flash_driver`。

附上`flash_driver`结构体内容：

``` cpp
struct flash_driver {
    const struct command_registration *commands;
    __FLASH_BANK_COMMAND((*flash_bank_command));
    int (*erase)(struct flash_bank *bank, unsigned int first, unsigned int last);
    int (*protect)(struct flash_bank *bank, int set, unsigned int first, unsigned int last);
    int (*write)(struct flash_bank *bank, const uint8_t *buffer, uint32_t offset, uint32_t count);
    int (*read)(struct flash_bank *bank, uint8_t *buffer, uint32_t offset, uint32_t count);

    int (*verify)(struct flash_bank *bank, const uint8_t *buffer, uint32_t offset, uint32_t count);
    int (*probe)(struct flash_bank *bank);
    int (*erase_check)(struct flash_bank *bank);
    int (*protect_check)(struct flash_bank *bank);
    int (*info)(struct flash_bank *bank, struct command_invocation *cmd);
    int (*auto_probe)(struct flash_bank *bank);
    void (*free_driver_priv)(struct flash_bank *bank);
};
```

一个`flash_driver`至少需要提供以下几个接口 *（这些接口如果没有实现，那也得保留一个return ERROR_OK，不然openocd会无法通过编译）*：

- `erase`：擦写bank的指定Sector区域；
- `protect`：开启/关闭对bank指定Sector区域的保护；
- `write`：从指定地址写入多个数据；
- `probe`：探测设备以确定Flash种类；
- `protect_check`：检查bank是否处于保护状态；
- `info`：显示Flash bank的信息；

部分接口可以自行编写或使用openocd提供的通用实现或复用已实现接口，如

- `read`可以使用openocd的`default_flash_read`；
- `erase_check`可以使用openocd的`default_flash_blank_check`；
- `free_driver_priv`可以使用openocd的`default_flash_free_driver_priv`；
- `auto_probe`可以复用`probe`；

> [!TIP]
> 这些接口都是提供给openocd的默认指令用的，
> 例如我们在使用`flash write_image`指令时，openocd就会调用到目标`flash_driver`的`.auto_probe`、`.probe`和`.write`函数。

## 适配中会使用到的接口

openocd提供了一套API让我们能操作设备的内存地址，常用的有：

- `target_read_u32`：从指定地址处中读取1个32位数据；
- `target_read_u16`：从指定地址处中读取1个16位数据；
- `target_read_u8`：从指定地址处中读取1个8位数据；
- `target_write_u32`：向指定地址处写入1个32位数据；
- `target_write_u16`：向指定地址处写入1个16位数据；
- `target_write_u8`：向指定地址处写入1个8位数据；
- `target_write_memory`：从指定地址处中读取n个32位数据；
- `target_read_memory`：向指定地址处写入n个32位数据；

可以使用上述API对设备寄存器进行读写以实现烧录。

## 附录

在openocd中可以尝试使用`target_write_memory`去写FLash，不过速度会很慢。

在openocd中有一种烧录方法：用一个单独的.C或.S文件设计一个Flash烧录算法，并将其编译成elf文件，再使用脚本bin2char.sh将elf文件中的
.text段抽成十六进制数放在.inc文件中 *（类似文件可以见`/contrib/loaders/flash/`目录中的内容）*，在进行烧录流程时分别将烧录算法与
待烧录数据加载进RAM中，再执行RAM中的烧录算法进行烧录。

### 多bank操作

如果设备上存在多段存储空间，我们可以通过`flash bank`指令分别为这些存储空间注册bank，
可以使用`flash banks`查看已注册的bank信息与数量。

### 多目标操作

如果设备拥有多颗核心，那么openocd就需要为每一颗核心设置一个目标（target），当openocd在启动一个拥有多个目标的cfg时，
如果没有指定端口号，openocd则会从3333端口号开始，逐一递增地为目标们开辟端口。

> [!IMPORTANT]
> 在为不同核心创建目标时，需要知道每个核心所使用的AP（Assert Port），并使用
> `-ap-num`为目标设置AP号，这样才能实现多核场景下对核心的单独控制。
>
> 如果没有指定`-ap-num`，那么所有的目标都会默认指向AP0，相当于多个目标控制
> 同一个核心。
>
> 核心的AP内容一般会在芯片手册的debug章节见到。

---
title: openocd适配新设备
published: 2025-04-01
description: '记录openocd适配新设备的一些知识'
image: ''
tags: ["嵌入式", "Openocd"]
category: 'Embedded'
draft: true
lang: ''
---

本文章记录的是我在研究openocd的新设备适配流程时所发现的一些规律。

## 适配流程

向openocd中添加新设备，就相当与向openocd中添加一个新的`flash_driver`。

一个`flash_driver`至少要手动实现以下几个接口 *（这些接口是必须得提供给openocd，哪怕内部实现仅仅是返回OK也行，不然openocd会无法正常运行）*：

- `erase`
- `protect`
- `write`
- `probe`
- `protect_check`
- `info`

而部分接口可以使用openocd提供的默认实现或是复用已实现的接口，如

- `read`可以使用openocd的`default_flash_read`
- `erase_check`可以使用openocd的`default_flash_blank_check`
- `free_driver_priv`可以使用openocd的`default_flash_free_driver_priv`
- `auto_probe`可以复用已实现的`probe`

> 在openocd中有一套由官方提供的默认指令集，openocd通过这套指令集实现对设备的擦除、烧录等操作，默认指令集是通过调用上述接口实现设备的擦除烧录的。

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

## `flash write_image` 的一些细节

我们在使用`flash write_image`指令进行烧录时，`flash write_image`指令会调用到`flash_driver`中的`.auto_probe`、`.probe`和`.write`进行程序烧录。

有时候我们

## 附录

在烧录时，openocd可以通过接口`target_write_u32`去一点一点写Flash，虽然这种方法会很直观，但烧录速度非常慢。

于是在openocd中还有另一种烧录方法：用一个单独的.C或.S文件设计一个Flash烧录算法，并将其编译成elf文件，再使用脚本bin2char.sh将elf文件中的.text段抽出来放在.inc文件中 *（类似文件可以见`/contrib/loaders/flash/`目录中的内容）*，在进行烧录流程时分别将烧录算法与待烧录数据加载进RAM中，再用烧录算法执行烧录， 这种方法会比使用`target_write_u32`去一点一点写Flash来的快。

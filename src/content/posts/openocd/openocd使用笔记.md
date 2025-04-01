---
title: openocd/openocd使用笔记
published: 2025-03-28
description: "记录一些openocd的基础操作."
image: ""
tags: ["嵌入式", "Openocd"]
category: Embedded
draft: true
lang: ''
---

# 使用openocd

最近我在研究开源调试器工具openocd时了解一些有趣的知识，写下本文章记录一下openocd的基础使用方法。

openocd作为一个命令行程序，它可以在命令行上运行，或者嵌入一些程序中使用，比如Eclipse或VSCode。

接下来我们假设一个想要用cmsis-dap去调试APM32F4x系列MCU的使用场景，我们将通过这个场景来了解openocd的一些基础用法：

## openocd与命令行

命令行是openocd的基础使用场景，要在命令行中运行openocd，首先需要告知openocd会使用到什么接口和目标，openocd是用cfg文件记录接口和
目标的，所以我们需要让openocd去加载我们要用到的接口和目标的cfg文件，

    openocd -f interface/cmsis-dap.cfg -f target/geehy/apm32f4x.cfg

指令启动openocd的gdb server服务。

> [!IMPORTANT]
> 在启动gdb server服务前请先确保你的APM32F4x系列MCU已经连接了电脑，否则gdb server服务将无法启动。

在启动gdb server服务后，openocd会开始监听以下端口:

- 3333: 提供给gdb连接
- 4444: 提供给telnet连接
- 6666: 提供给tcl连接

> [!TIP]
> 如果想要调试程序，那就要用gdb，如果只想要烧录擦除，那用telnet即可。

现在，假设我们需要调试程序为`test.elf`，其路径在`D:/workspace`中，现在我们想要将它烧录进MCU中并进行调试，那可以使用gdb。

### gdb

如果要使用gdb去调试程序，那么我们需要先让gdb加载elf文件（如果命令行当前所在路径为`D:/workspace`上时，可以省略`D:/workspace`，
只用输入`test.elf`）：

    arm-none-eabi-gdb D:/workspace/test.elf

执行完指令后会进入到gdb的命令行界面，

> [!NOTE]
> **为什么要使用elf文件?**
> > 因为gdb需要有debug信息才能进行跟踪源码，而一般来说，只有elf中才会记录debug信息，hex或bin文件则不会记录

接着，让gdb去连接openocd，

> 别记错了，openocd提供给gdb的端口可是3333哦ˋ( ° ▽、° )

    target remote localhost:3333

接着，停住MCU，

    monitor halt

执行烧录与擦除操作，

    monitor flash write_image erase test.elf

烧录完成后，复位MCU并让MCU运行，

    monitor reset
    continue

> 如果需要让程序在mian函数开头停住，可以在`continue`指令前加上`break main`

#### gdb在调试时的一些常用指令

| 指令                | 作用                             |
| ------------------- | -------------------------------- |
| list                | 列出当前所在行及其以下的10行代码   |
| list function       | 列出函数function的代码           |
| next                | step over 逐句跳过               |
| step                | step into 逐句进入               |
| finish              | step over 跳出                   |
| continue            | 全速运行，直到结束或碰到断点      |
| break filename:line | 在filename的line行添加断点       |
| break function      | 在函数function处添加断点         |
| info breakpoints    | 查看所有断点                     |
| delete breakpoints  | 删除所有断点                     |

如果只是想要烧录但不调试，那么可以使用telnet。

### telnet

telnet一般用于只烧录不调试的场景。

先打开telnet并连接上openocd

> openocd开放给telnet的端口是4444哦

    telnet localhost 4444

接着暂停MCU运行并接擦除烧录程序

    halt
    flash write_image erase test.elf

烧录完成后，复位MCU并让MCU运行:

    reset run

---
---

## openocd与Eclipse

Eclipse封装了openocd + gdb的调试流程，可以通过Eclipse的配置界面调整参数以控制烧录流程。

- **OpenOCD Path**

首先，需要告知Eclipse openocd的安装路径，点击Eclipse菜单栏中的`Windows`->`Preferences`，在弹出的窗口中选择
`Global OpenOCD Path`，向Folder填入openocd的安装路径，点击`Apply and Close`应用配置并关闭窗口

> [!TIP]
> 完成这步后我们成功解锁`${openocd_path}`通配符，使用`${openocd_path}`通配符时将直接指向openocd路径。

![OpenOCD Path](/src/content/posts/openocd/Openocd%20Path.png)

接下来得开始配置Debug，打开`Debug Configuration`，双击`GDB OpenOCD Debugging`新建Debug配置，

![Debug Configuration](/src/content/posts/openocd/Debug%20Configuration.png)

选中新建的Debug配置后，可以发现右侧面板中多了几个页面，包括`Main`，`Debugger`，`Startup`等，在这些页面中我们目前需要关注的是
`Main`、`Debugger`页。

- **Main**

向`C/C++ Application`需要填入elf文件的地址，可以是绝对路径或相对路径，相对路径的起始地址为项目的根目录（也可以使用
`${project_loc}`、`${project_name}`等通配符代替一部分路径或文件名）。

> [!TIP]
> 如果项目是先完成了编译再新建，那么`C/C++ Application`有可能会由Eclipse自动填充。
> > `${project_loc}`表示当前项目所在的目录的绝对路径，`${project_name}`表示当前项目名称

![Main]()

- **Debugger**

在`OpenOCD Setup`->`Config options`中加入openocd加载cfg文件的命令

    -f interface/cmsis-dap.cfg -f target/geehy/apm32f4x.cfg

将`GDB Client Setup`->`Executable name`中的gdb改为arm-none-eabi-gdb

完成上述步骤后，点击Debug按钮开始调试，Eclipse会在完成烧录动作后进入到调试界面。

### Eclipse附录

我们在使用Eclipse调试时，会经常用到`Debug Configuration`中的`Main`、`Debugger`和`Startup`页去控制烧录流程，下面简单介绍一下这些
页中的内容。

#### Main

#### Debugger

#### Startup

---
layout:     post
title:      Uboot-2 新增板子支持
subtitle:   
date:       2017-08-15
author:     ZGB
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Uboot
---

## 介绍
上一篇文章讲述了uboot的编译和配置，这里接着看看新增一块板子需要做的事情。

对于每块板子来说，它的外设可能不一样，架构可能不一样，芯片也有一些特殊的性质，但是一些通用外设的驱动，整个启动框架都是一样的，所以uboot提供一套代码框架给我们，我们在增加一块板子的时候，按照uboot的机制来增加，可以减少很多后期的麻烦处理。

对于一块板子来说，最大的不同就是板级文件的不同，这个板级文件一般是一个c文件，需要实现一些特定的功能。还需要一个.h来对应这个板级文件。然后编译的时候，我们需要一个defconfig来区分是那块板子。

1.在board/下创建目录myvendor/

2.在myvendor/目录下新建目录myboard/

3.在myboard/目录下创建board.c、board.h、Kconfig、Makefile

```
## Kconfig
if TARGET_MY_AM43XX

config SYS_BOARD
	default "myboard"

config SYS_VENDOR
	default "myvendor"

config SYS_SOC
	default "mysoc"

config SYS_CONFIG_NAME
	default "myconfig"

endif
```

```
## Makefile
obj-y	+= board.o
```

4.在include\configs目录下创建myconfig.h

5.对于TI，还要将board\ti\common文件夹复制到board\myvendor\目录下

6.在configs\目录下创建文件my_defconfig
```
CONFIG_ARM=y
CONFIG_TARGET_MY_AM43XX=y
CONFIG_SPL=y
CONFIG_SYS_EXTRA_OPTIONS="SERIAL1,CONS_INDEX=1,BOARD_SBC_EC8800"
# CONFIG_CMD_IMLS is not set
# CONFIG_CMD_FLASH is not set
# CONFIG_CMD_SETEXPR is not set
CONFIG_SPI_FLASH_BAR=y
CONFIG_SPI_FLASH=y
```
7.在arch\arm\Kconfig中，增加

```
config TARGET_MY_AM43XX
       bool "Support my_am43xx"
       select CPU_V7
       select SUPPORT_SPL
       select CREATE_BOARD_SYMLINK
      
source "board/myvendor/myboard/Kconfig"
```
8.在include\configs\myconfig.h中加入
```

```




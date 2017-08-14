---
layout:     post
title:      Uboot-1 Uboot编译配置
subtitle:   
date:       2017-08-14
author:     ZGB
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Uboot
---

## 介绍
在进入zImage之前，需要一段引导程序来引导内核启动，即BootLoader，对于Linux而言，就是熟悉的U-boot。说他熟悉，可能只是主观上感觉比较亲切，一旦对它深入研究起来，并不像想象中的那么温和。不管它温柔的外表下隐藏着什么样的玄机，我都会尝试着去理解和剖析uboot，希望能够通过此次探索对uboot有更多的理解。

## 编译uboot的过程
编译uboot的过程并不复杂，如果安装好了虚拟机，或者有现成的Linux环境，那么你可以从git上下载其源代码。
```
git clone http://git.denx.de/?p=u-boot.git;a=summary
```
如果想了解更多信息，可以看看官方wiki：
> http://www.denx.de/wiki/U-Boot/

有了源代码，还需一个编译器，编译器的下载这里就不说了。有了编译之后，可以选择将编译器的路径和架构加入到环境变量中，方便编译。

接下来就可以简单粗暴的操作了。
```
make xxx_defconfig
make
```
这样就会在uboot的根目录生成我们需要的文件。

大多数情况下，`xxx_defconfig`的选用原则，就是在`configs/`目录下面找一个最接近的板子，如果我们选用官方的板子，那么这个目录下面，基本上都能找到与板子对应的defconfig。

## make xxx_defconfig
这个命令最终会在uboot根目录下生成`.config`文件。为了细致的理解这个步骤具体做了啥，我以am43xx_evm_defconfig这个为例，看看具体发生了什么。

在顶层Makefile中，找到下面的描述。
```mk
%config: scripts_basic outputmakefile FORCE
	$(Q)$(MAKE) $(build)=scripts/kconfig $@
```
这里对应的就是我们的`xxx_defconfig`文件，%是通配符。这里先看看对应的几个依赖。
##### scripts_basic
```
scripts_basic:
	$(Q)$(MAKE) $(build)=scripts/basic
	$(Q)rm -f .tmp_quiet_recordmcount
```
在顶层Makefile中：
```
include scripts/Kbuild.include
```
这个文件定义了build：
```mk
# Shorthand for $(Q)$(MAKE) -f scripts/Makefile.build obj=
# Usage:
# $(Q)$(MAKE) $(build)=dir
build := -f $(srctree)/scripts/Makefile.build obj
```
展开scripts_basic就是：
```
make -f scripts/Makefile.build obj=scripts/basic
rm -f .tmp_quiet_recordmcount
```
意思是根据scripts/Makefile.build中的规则来构建默认的目标，这个目标是scripts/Makefile.build中的__build，并传入参数obj=scripts/basic，最终生成了主机用的程序fixdep，这个程序用来处理依赖文件。

*注：关于build的构建规则详情请参考附录一《**build构建规则**》。关于主机程序fixdep的更多信息请参考附录二《**fixdep**》。*

##### outputmakefile
```mk
# outputmakefile generates a Makefile in the output directory, if using a
# separate output directory. This allows convenient use of make in the
# output directory.
outputmakefile:
ifneq ($(KBUILD_SRC),)
	$(Q)ln -fsn $(srctree) source
	$(Q)$(CONFIG_SHELL) $(srctree)/scripts/mkmakefile \
	    $(srctree) $(objtree) $(VERSION) $(PATCHLEVEL)
endif
```
这里用来在输出目录生成一个Makefile，方便使用make。主要用在需要指定编译输出文件路径的时候，如果不指定，默认是在uboot顶层目录生成。可以通过下面的命令指定输出目录。
```sh
export KBUILD_OUTPUT=dir/to/store/output/files/
```

##### FORCE
```mk
PHONY += FORCE
FORCE:

# Declare the contents of the .PHONY variable as phony.  We keep that
# information in a variable so we can use it in if_changed and friends.
.PHONY: $(PHONY)
```
可以看出，FORCE中记录了所有伪目标的信息。可以供其他的地方调用。还有一点就是根据伪目标不管更不更新都会执行的特性，如果一个目标添加了依赖FORCE，将会确保每次make的时候都会执行。

##### %config
所有的依赖都过了一遍，可以看出这些依赖只是做了些准备工作，那么，继续看看展开后的`%config`目标。
```mk
make -f scripts/Makefile.build obj=scripts/kconfig am43xx_evm_defconfig
```
但是，am43xx_evm_defconfig并没有在scripts/Makefile.build中定义，而是在scripts/Makefile.build中
```mk
# The filename Kbuild has precedence over Makefile
kbuild-dir := $(if $(filter /%,$(src)),$(src),$(srctree)/$(src))
kbuild-file := $(if $(wildcard $(kbuild-dir)/Kbuild),$(kbuild-dir)/Kbuild,$(kbuild-dir)/Makefile)
include $(kbuild-file)
```
这里的`$(src)`被赋值为`$(obj)`，即`scripts/kconfig`，`srctree`被赋值为`.`。

所以，这两句话的意思是，如果`$(src)`中含有字符`/`，则将`kbuild-dir`赋值为`$(srctree)/$(src)`，即`./scripts/kconfig`。如果`$(kbuild-dir)/Kbuild`这个目标存在，则将`kbuild-file`赋值为`$(kbuild-dir)/Makefile`，即`./scripts/kconfig/Makefile`。

最后一句话`include $(kbuild-file)`，即`include ./scripts/kconfig/Makefile`。在这个Makefile中，
```mk
%_defconfig: $(obj)/conf
	$(Q)$< --defconfig=arch/$(SRCARCH)/configs/$@ $(Kconfig)
```
可以看出xxx_defconfig依赖于scripts/kconfig/conf。
关于scripts/kconfig/conf的生成，有人说是Makefile会自动推导依赖，会自动去编译conf.c。但是，在目录scripts/kconfig/去查看的时候，发现还编译了zconf.tab.o，自动推导就无法解释这个文件的由来了。

在scripts/kconfig/Makefile中：
```mk
conf-objs	:= conf.o  zconf.tab.o
```
根据Makefile的语法，这句话的意思是conf目标由conf.c、zconf.tab.c编译成conf.o、zconf.tab.o后链接而来。这里就解释了conf的由来，conf主要用于生成.config文件，具体的过程可以研究下这两个.c文件的内容。

conf文件的由来已经搞清楚了，现在%_defconfig展开后的结果为：
```mk
am43xx_evm_defconfig:
    scripts/kconfig/conf --defconfig=arch/../configs/am43xx_evm_defconfig Kconfig
```
scripts/kconfig/conf会根据传入的参数，最终生成.config文件，具体的过程请参考附录三《**conf**》。

至此，make xxx_defconfig的过程就介绍完了。

## Make
在执行完`make xxx_defconfig`后，如果还有什么地方是一定要手动修改下的，可以继续执行`make menuconfig`修改。如果没什么需要修改的，可以直接`make`来生成uboot文件了。

在顶层Makefile中，找到默认的目标。
```mk
# That's our default target when none is given on the command line
PHONY := _all
_all:
```
在接下来的代码中会补充`_all`的依赖。
```mk
# If building an external module we do not care about the all: rule
# but instead _all depend on modules
PHONY += all
ifeq ($(KBUILD_EXTMOD),)
_all: all
else
_all: modules
endif
```
这里的KBUILD_EXTMOD为空，应为没有设置`M`和`SUBDIRS`（我目前没找到哪设置了，感觉没有设置）。
```mk
# Use make M=dir to specify directory of external module to build
# Old syntax make ... SUBDIRS=$PWD is still supported
# Setting the environment variable KBUILD_EXTMOD take precedence
ifdef SUBDIRS
  KBUILD_EXTMOD ?= $(SUBDIRS)
endif

ifeq ("$(origin M)", "command line")
  KBUILD_EXTMOD := $(M)
endif
```
于是，`_all`目标就依赖于`all`。
```mk
all:		$(ALL-y)
```
接着看`$(ALL-y)`被赋值成了什么。
```mk
# Always append ALL so that arch config.mk's can add custom ones
ALL-y += u-boot.srec u-boot.bin System.map u-boot.cfg binary_size_check

ALL-$(CONFIG_ONENAND_U_BOOT) += u-boot-onenand.bin
ifeq ($(CONFIG_SPL_FSL_PBL),y)
ALL-$(CONFIG_RAMBOOT_PBL) += u-boot-with-spl-pbl.bin
else
ALL-$(CONFIG_RAMBOOT_PBL) += u-boot.pbl
endif
ALL-$(CONFIG_SPL) += spl/u-boot-spl.bin
ALL-$(CONFIG_SPL_FRAMEWORK) += u-boot.img
ALL-$(CONFIG_TPL) += tpl/u-boot-tpl.bin
ALL-$(CONFIG_OF_SEPARATE) += u-boot.dtb u-boot-dtb.bin
ifeq ($(CONFIG_SPL_FRAMEWORK),y)
ALL-$(CONFIG_OF_SEPARATE) += u-boot-dtb.img
endif
ALL-$(CONFIG_OF_HOSTFILE) += u-boot.dtb
ifneq ($(CONFIG_SPL_TARGET),)
ALL-$(CONFIG_SPL) += $(CONFIG_SPL_TARGET:"%"=%)
endif
ALL-$(CONFIG_REMAKE_ELF) += u-boot.elf

ifneq ($(BUILD_ROM),)
ALL-$(CONFIG_X86_RESET_VECTOR) += u-boot.rom
endif

# enable combined SPL/u-boot/dtb rules for tegra
ifneq ($(CONFIG_TEGRA),)
ifeq ($(CONFIG_SPL),y)
ifeq ($(CONFIG_OF_SEPARATE),y)
ALL-y += u-boot-dtb-tegra.bin
else
ALL-y += u-boot-nodtb-tegra.bin
endif
endif
endif

# Add optional build target if defined in board/cpu/soc headers
ifneq ($(CONFIG_BUILD_TARGET),)
ALL-y += $(CONFIG_BUILD_TARGET:"%"=%)
endif

LDFLAGS_u-boot += $(LDFLAGS_FINAL)
ifneq ($(CONFIG_SYS_TEXT_BASE),)
LDFLAGS_u-boot += -Ttext $(CONFIG_SYS_TEXT_BASE)
endif
```
可以看出ALL-y的值有很多，这里选择几个重要的分析下。首先是默认就被赋值的参数。
```mk
ALL-y += u-boot.srec u-boot.bin System.map u-boot.cfg binary_size_check
```
这几个文件是一定会生成的。然后关注SPL的选项。
```mk
ALL-$(CONFIG_SPL) += spl/u-boot-spl.bin
```
如果在.config中有`CONFIG_SPL=y`，那么就会生成u-boot-spl.bin，即uboot的stage1。我们先看看这个文件的生成。

### SPL
查找u-boot-spl.bin目标。
```mk
spl/u-boot-spl.bin: spl/u-boot-spl
	@:
```
这里`@`表示不回显，`:`代表不做任何的操作，这里的`spl/u-boot-spl.bin`仅仅依赖于`spl/u-boot-spl`。
```mk
spl/u-boot-spl: tools prepare
	$(Q)$(MAKE) obj=spl -f $(srctree)/scripts/Makefile.spl all
```
先看看这里的两个依赖。tools和prepare。
```mk
tools: prepare
```
可见最后都是依赖了prepare。
```mk
# All the preparing..
prepare: prepare0
```
整个perpare的依赖关系是一层接一层的，我们一层层的看都做了些啥。
```mk
archprepare: prepare1 scripts_basic

prepare0: archprepare FORCE
	$(Q)$(MAKE) $(build)=.
```
这里又遇见了scripts_basic，首先创建了主机程序fixdep。
```mk
prepare1: prepare2 $(version_h) $(timestamp_h) \
                   include/config/auto.conf
ifeq ($(CONFIG_HAVE_GENERIC_BOARD),)
ifeq ($(CONFIG_SYS_GENERIC_BOARD),y)
	@echo >&2 "  Your architecture does not support generic board."
	@echo >&2 "  Please undefine CONFIG_SYS_GENERIC_BOARD in your board config file."
	@/bin/false
endif
endif
ifeq ($(wildcard $(LDSCRIPT)),)
	@echo >&2 "  Could not find linker script."
	@/bin/false
endif
```
prepare1对一些变量进行检查。
```mk
# prepare2 creates a makefile if using a separate output directory
prepare2: prepare3 outputmakefile
```
这里又遇见了outputmakefile，可见前期准备的东西和make xxx_defconfig的时候差不多。
```mk
# prepare3 is used to check if we are building in a separate output directory,
# and if so do:
# 1) Check that make has not been executed in the kernel src $(srctree)
prepare3: include/config/uboot.release
ifneq ($(KBUILD_SRC),)
	@$(kecho) '  Using $(srctree) as source for U-Boot'
	$(Q)if [ -f $(srctree)/.config -o -d $(srctree)/include/config ]; then \
		echo >&2 "  $(srctree) is not clean, please run 'make mrproper'"; \
		echo >&2 "  in the '$(srctree)' directory.";\
		/bin/false; \
	fi;
endif
```
prepare3基本不会用到，因为我们没有指定最后的输出目录。

可以看出，prepare做了一些准备工作，这些工作和make xxx_defconfig的时候所做的工作相差无几。

分析完了依赖，接着看命令部分。
```mk
$(Q)$(MAKE) obj=spl -f $(srctree)/scripts/Makefile.spl all
```
`-f`参数指定了构建`all`目标的Makefile。我们在scripts/Makefile.spl中去找all目标。
```mk
ALL-y	+= $(obj)/$(SPL_BIN).bin $(obj)/$(SPL_BIN).cfg

ifdef CONFIG_SAMSUNG
ALL-y	+= $(obj)/$(BOARD)-spl.bin
endif

ifdef CONFIG_SUNXI
ALL-y	+= $(obj)/sunxi-spl.bin
endif

ifeq ($(CONFIG_SYS_SOC),"at91")
ALL-y	+= boot.bin
endif

all:	$(ALL-y)
```
这里有几个特殊的判断条件，比较容易懂，是针对几种特殊平台的，暂时不管。看看主要的部分。
```mk
ALL-y	+= $(obj)/$(SPL_BIN).bin $(obj)/$(SPL_BIN).cfg
```
这里`$(obj)`就是刚才传入的参数`spl`，`$(SPL_BIN)`被赋值为`u-boot-spl`，整句话翻译过来就是。
```mk
ALL-y	+= spl/u-boot-spl.bin spl/u-boot-spl.cfg
```
这里就会生成我们熟悉的SPL文件。继续看看spl/u-boot-spl.bin怎么来的。
```mk
$(obj)/$(SPL_BIN).bin: $(obj)/$(SPL_BIN) FORCE
	$(call if_changed,objcopy)
```
它依赖于spl/u-boot-spl。
```mk
$(obj)/$(SPL_BIN): $(u-boot-spl-init) $(u-boot-spl-main) $(obj)/u-boot-spl.lds FORCE
	$(call if_changed,u-boot-spl)
```
这里的u-boot-spl.lds是链接文件，决定入口代码。剩下的u-boot-spl-init和u-boot-spl-main决定了哪些代码将被编译。
```mk
u-boot-spl-init := $(head-y)
u-boot-spl-main := $(libs-y)
```
我们找到head-y的定义。
```mk
head-y		:= $(addprefix $(obj)/,$(head-y))
```
这句话的意思是在head-y本身前面加上前缀$(obj)，即spl，但是这里并没有真正对head-y赋值。那么需要继续寻找head-y。在scripts/Makefile.spl中
```mk
include $(srctree)/config.mk
include $(srctree)/arch/$(ARCH)/Makefile
```
这里的ARCH被赋值为arm，所以这里包含了“../arch/arm/Makefile”。在这个文件中，我们找到
```mk
head-y := arch/arm/cpu/$(CPU)/start.o
```
在include/config/auto.conf中，CPU的值为armv7。这里就找到了第一个要编译的文件，`arch/arm/cpu/armv7/start.o`。
我们继续看看libs-y。在scripts/Makefile.spl和arch/arm/Makefile中，都有定义。
```mk
#arch/arm/Makefile
libs-y += arch/arm/cpu/$(CPU)/
libs-y += arch/arm/cpu/
libs-y += arch/arm/lib/

ifeq ($(CONFIG_SPL_BUILD),y)
ifneq (,$(CONFIG_MX23)$(CONFIG_MX35)$(filter $(SOC), mx25 mx27 mx5 mx6 mx31 mx35))
libs-y += arch/arm/imx-common/
endif
else
ifneq (,$(filter $(SOC), mx25 mx27 mx5 mx6 mx31 mx35 mxs vf610))
libs-y += arch/arm/imx-common/
endif
endif

ifneq (,$(filter $(SOC), kirkwood))
libs-y += arch/arm/mach-mvebu/
endif
```
```mk
#scripts/Makefile.spl
libs-y += $(if $(BOARDDIR),board/$(BOARDDIR)/)
libs-$(HAVE_VENDOR_COMMON_LIB) += board/$(VENDOR)/common/

libs-$(CONFIG_SPL_FRAMEWORK) += common/spl/
libs-$(CONFIG_SPL_LIBCOMMON_SUPPORT) += common/
libs-$(CONFIG_SPL_LIBDISK_SUPPORT) += disk/
libs-$(CONFIG_SPL_DM) += drivers/core/
libs-$(CONFIG_SPL_I2C_SUPPORT) += drivers/i2c/
libs-$(CONFIG_SPL_GPIO_SUPPORT) += drivers/gpio/
libs-$(CONFIG_SPL_MMC_SUPPORT) += drivers/mmc/
libs-$(CONFIG_SPL_MPC8XXX_INIT_DDR_SUPPORT) += drivers/ddr/fsl/
libs-$(CONFIG_SYS_MVEBU_DDR) += drivers/ddr/mvebu/
libs-$(CONFIG_SPL_SERIAL_SUPPORT) += drivers/serial/
libs-$(CONFIG_SPL_SPI_FLASH_SUPPORT) += drivers/mtd/spi/
libs-$(CONFIG_SPL_SPI_SUPPORT) += drivers/spi/
libs-y += fs/
libs-$(CONFIG_SPL_LIBGENERIC_SUPPORT) += lib/
libs-$(CONFIG_SPL_POWER_SUPPORT) += drivers/power/ drivers/power/pmic/
libs-$(CONFIG_SPL_MTD_SUPPORT) += drivers/mtd/
libs-$(CONFIG_SPL_NAND_SUPPORT) += drivers/mtd/nand/
libs-$(CONFIG_SPL_DRIVERS_MISC_SUPPORT) += drivers/misc/
libs-$(CONFIG_SPL_ONENAND_SUPPORT) += drivers/mtd/onenand/
libs-$(CONFIG_SPL_DMA_SUPPORT) += drivers/dma/
libs-$(CONFIG_SPL_POST_MEM_SUPPORT) += post/drivers/
libs-$(CONFIG_SPL_NET_SUPPORT) += net/
libs-$(CONFIG_SPL_ETH_SUPPORT) += drivers/net/
libs-$(CONFIG_SPL_ETH_SUPPORT) += drivers/net/phy/
libs-$(CONFIG_SPL_USBETH_SUPPORT) += drivers/net/phy/
libs-$(CONFIG_SPL_MUSB_NEW_SUPPORT) += drivers/usb/musb-new/
libs-$(CONFIG_USB_DWC3_GADGET) += drivers/usb/dwc3/
libs-$(CONFIG_USB_DWC3_GADGET) += drivers/usb/gadget/udc/
libs-$(CONFIG_SPL_USBETH_SUPPORT) += drivers/usb/gadget/
libs-$(CONFIG_SPL_WATCHDOG_SUPPORT) += drivers/watchdog/
libs-$(CONFIG_SPL_USB_HOST_SUPPORT) += drivers/usb/host/
libs-$(CONFIG_OMAP_USB_PHY) += drivers/usb/phy/
libs-$(CONFIG_SPL_SATA_SUPPORT) += drivers/block/

head-y		:= $(addprefix $(obj)/,$(head-y))
libs-y		:= $(addprefix $(obj)/,$(libs-y))

libs-y := $(patsubst %/, %/built-in.o, $(libs-y))
```
最后两句话的意思是将libs-y加上$(obj)/，再将后缀/替换为/built-in.o。即所有的libs-y加上了`spl/build-in.o`后缀。即之前的`libs-y = drivers/net/ ···`，变换之后的`libs-y = drivers/net/spl/build-in.o ···`。而每个build-in.o都是由所在目录的Makefile生成的，我们在这个目录下面去寻找Makefile中的内容就能知道哪些代码被编译了。


### u-boot
接下来我们看看uboot的通用目标。
```mk
ALL-y += u-boot.srec u-boot.bin System.map u-boot.cfg binary_size_check
```
1. u-boot.srec
```mk
u-boot.hex u-boot.srec: u-boot FORCE
	$(call if_changed,objcopy)
```

2. u-boot.bin
```mk
u-boot.bin: u-boot FORCE
	$(call if_changed,objcopy)
	$(call DO_STATIC_RELA,$<,$@,$(CONFIG_SYS_TEXT_BASE))
	$(BOARD_SIZE_CHECK)
```

3. System.map
```mk
SYSTEM_MAP = \
		$(NM) $1 | \
		grep -v '\(compiled\)\|\(\.o$$\)\|\( [aUw] \)\|\(\.\.ng$$\)\|\(LASH[RL]DI\)' | \
		LC_ALL=C sort
System.map:	u-boot
		@$(call SYSTEM_MAP,$<) > $@
```

4. u-boot.cfg
```mk
u-boot.cfg:	include/config.h
	$(call if_changed,cpp_cfg)
```

5. binary_size_check
```mk
binary_size_check: u-boot.bin FORCE
	@file_size=$(shell wc -c u-boot.bin | awk '{print $$1}') ; \
	map_size=$(shell cat u-boot.map | \
		awk '/_image_copy_start/ {start = $$1} /_image_binary_end/ {end = $$1} END {if (start != "" && end != "") print "ibase=16; " toupper(end) " - " toupper(start)}' \
		| sed 's/0X//g' \
		| bc); \
	if [ "" != "$$map_size" ]; then \
		if test $$map_size -ne $$file_size; then \
			echo "u-boot.map shows a binary size of $$map_size" >&2 ; \
			echo "  but u-boot.bin shows $$file_size" >&2 ; \
			exit 1; \
		fi \
	fi
```

这里最主要的依赖就是uboot，我们重点看看这个依赖。
```mk
u-boot:	$(u-boot-init) $(u-boot-main) u-boot.lds
	$(call if_changed,u-boot__)
ifeq ($(CONFIG_KALLSYMS),y)
	$(call cmd,smap)
	$(call cmd,u-boot__) common/system_map.o
endif
```
这里的依赖和SPL阶段就很像了。
```mk
libs-y += lib/
libs-$(HAVE_VENDOR_COMMON_LIB) += board/$(VENDOR)/common/
libs-$(CONFIG_OF_EMBED) += dts/
libs-y += fs/
libs-y += net/
libs-y += disk/
libs-y += drivers/
libs-y += drivers/dma/
libs-y += drivers/gpio/
libs-y += drivers/i2c/
libs-y += drivers/mmc/
libs-y += drivers/mtd/
libs-$(CONFIG_CMD_NAND) += drivers/mtd/nand/
libs-y += drivers/mtd/onenand/
libs-$(CONFIG_CMD_UBI) += drivers/mtd/ubi/
libs-y += drivers/mtd/spi/
libs-y += drivers/net/
libs-y += drivers/net/phy/
libs-y += drivers/pci/
libs-y += drivers/power/ \
	drivers/power/fuel_gauge/ \
	drivers/power/mfd/ \
	drivers/power/pmic/ \
	drivers/power/battery/ \
	drivers/power/regulator/
libs-y += drivers/spi/
libs-$(CONFIG_FMAN_ENET) += drivers/net/fm/
libs-$(CONFIG_SYS_FSL_DDR) += drivers/ddr/fsl/
libs-y += drivers/serial/
libs-y += drivers/usb/dwc3/
libs-y += drivers/usb/emul/
libs-y += drivers/usb/eth/
libs-y += drivers/usb/gadget/
libs-y += drivers/usb/gadget/udc/
libs-y += drivers/usb/host/
libs-y += drivers/usb/musb/
libs-y += drivers/usb/musb-new/
libs-y += drivers/usb/phy/
libs-y += drivers/usb/ulpi/
libs-y += common/
libs-$(CONFIG_API) += api/
libs-$(CONFIG_HAS_POST) += post/
libs-y += test/
libs-y += test/dm/
libs-$(CONFIG_UT_ENV) += test/env/

libs-y += $(if $(BOARDDIR),board/$(BOARDDIR)/)

libs-y := $(sort $(libs-y))

u-boot-dirs	:= $(patsubst %/,%,$(filter %/, $(libs-y))) tools examples

u-boot-alldirs	:= $(sort $(u-boot-dirs) $(patsubst %/,%,$(filter %/, $(libs-))))

libs-y		:= $(patsubst %/, %/built-in.o, $(libs-y))

u-boot-init := $(head-y)
u-boot-main := $(libs-y)
```
根据SPL阶段的分析，libs-y为各目录下的built-in.o集合。各个目录下的Makefile会将编译的.o文件合成built-in.o，所以这里就包含了所有的.o文件。head-y和SPL一样，都指向了arch/arm/cpu/$(CPU)/start.o。其他的地方和SPL阶段大同小异，这里就不细说了。

### MLO
在arch\arm\cpu\armv7\am33xx\config.mk中，定义了MLO和u-boot.img。
```mk
ifdef CONFIG_SPL_BUILD
ALL-y	+= MLO
ALL-$(CONFIG_SPL_SPI_SUPPORT) += MLO.byteswap
else
ALL-y	+= u-boot.img
endif
```

## 附录一
### build构建规则
本文摘自网友博客。
#### 一. build定义：
```mk
scripts/Kbuild.include
build := -f $(if $(KBUILD_SRC),$(srctree)/)scripts/Makefile.build obj
```
`$(KBUILD_SRC)`常规情况下为空，所以变量定义可简化为：
```
build := -f scripts/Makefile.build obj
```

#### 二. (MAKE) $(build)=的处理过程
build使用的一般形式为：
> $(MAKE) $(build)=*build_dir*  *[argv]*

斜体字部分为可变目录和参数，其中[argv] 可选。使用`scripts/Kbuild.include`中的`$(build)`变量定义，进行变量替换后，上述命令则为：
```
$(MAKE) -f scripts/Makefile.build obj=build_dir  [argv]
```
Make进入由参数-f指定的Make文件`scripts/Makefile.build`，并传入参数`obj=build_dir` 和`argv`。

在`scripts/Makefile.build`的处理过程中，传入的参数`$(obj)` 代表此次Make命令要处理（编译、链接、和生成） 文件所在的目录，该目录下通常情况下都会存在的`Makefile`文件会被`Makefile.build`包含。`$(obj)`目录下的`Makefile`记为`$(obj)/Makefile`。针对Make命令，有两种情况：不指明Make目标和指定目标。
##### 1.不指定目标
```
$(MAKE) $(build)=build_dir  [argv]
```
中，当没有参数[argv]时，该Make命令没有指定目标。如顶层Makefile中，$(vmlinux-dirs)的构建规则：
```
$(vmlinux-dirs): prepare scripts
     $(Q)$(MAKE) $(build)=$@
```
其他的还有主机程序fixdep的构建规则：
```
scripts_basic:
    $(Q)$(MAKE) $(build)=scripts/basic
```
这时会使用`Makefile.build`中的默认目标`__build`。然后更进一步，会使用`$(obj)/Makefile`中定义的变量来进行目标匹配。

`__build`在`Makefile.build`中的构建规则为：
```
__build:$(if $(KBUILD_BUILTIN),$(builtin-target) $(lib-target) $(extra-y))\
    $(if $(KBUILD_MODULES),$(obj-m) $(modorder-target))\
    $(subdir-ym)$(always)
    @:
```
首先会构建该默认目标`__build`的依赖。Make会寻找重建这些依赖的规则。而这些规则要么在当前的Makefile文件`Makefile.build`中，要么在`Makefile.build` include的`$(obj)/Makefile`中。在此不指定Make目标的情况下，会使用`Makefile.build`中的构建规则来重建依赖。相应地，只重构那些在`$(obj)/Makefile`定义的依赖。其他的依赖要么不满足条件，要么找不到重构规则而被忽略。

例如上述`$(vmlinux-dirs)`规则：
```
$(vmlinux-dirs): prepare scripts
     $(Q)$(MAKE) $(build)=$@
```
的构建中，没有指明Make目标，那么将使用`Makefile.build`中的默认目标`__build`，且会包含上述多目标`$(vmlinux-dirs)`中，实际处理时的某单一目标（此为目录）下的Makefile文件，即`$(obj)/Makefile`。在`__build`规则中，因`$(KBUILD_BUILTIN)`被主目录设置为1，且export，所以将首先重建依赖`$(builtin-target)`。而依赖`$(builtin-target)`的重建规则在`Makefile.build`中，即：
```mk
$(builtin-target):$(obj-y)FORCE  ####$(obj)/built-in.o,且先要构建依赖$(obj-y)
    $(call if_changed,link_o_target)
```
该规则同样要首先重建依赖`$(obj-y)`。而`$(obj-y)`在`$(obj)/Makefile`中定义且被赋值。这时Make又会查找`$(obj-y)`包含文件的构建规则。同样地，该规则要么在`Makefile.build`中，要么在`$(obj)/Makefile`中。此处`$(obj-y)`为.o文件的合集，.o文件的构建规则在`Makefile.build`中被定义：
```
$(obj)/%.o:$(src)/%.c$(recordmcount_source)FORCE  ##$(obj-y)匹配规则
    $(call cmd,force_checksrc)
    $(call if_changed_rule,cc_o_c)
```
就是重复这样一个递归的“规则----> 规则目标-->依赖--->重建依赖--->规则---->”....的过程，直至最后的目标文件被构建，然后逆推，由依赖层层重建其规则目标。如果构建的最终目标是主机程序，如上述fixdep的构建规则 ：
```
scripts_basic:
    $(Q)$(MAKE) $(build)=scripts/basic
```
`Makefile.build`除了要inclue `$(obj)/Makefile`即`scripts/basic/Makefile`文件外，还会include `scripts/Makefile.host`，在`Makefile.build` 对包含`scripts/Makefile.host`的处理如下：
```
ifneq ($(hostprogs-y)$(hostprogs-m),)
include scripts/Makefile.host   #编译主机程序时，要包含Makefile.host文件
endif
```
因为在此之前已经包含了`scripts/basic/Makefile`，且在该Makefile文件里，`hostprogs-y`被赋值为`fixdep`，那么上述`ifneq`分支为真。即会包含`scripts/Makefile.host`。

在`scripts/basic/Makefile`里，有如下语句：
```
always        := $(hostprogs-y)
```
那么回到不指定目标的make命令里，接下来在`Makefile.build`中的默认目标`__build`中，会匹配到重建的依赖`always`。注意这里变量`$(always)`的值为`fixdep`。由此触发依赖`fixdep`的重建（否则，就没有入口目标的依赖层层重建至fixdep）。而它的重建规则在上面包含的`scripts/Makefile.host`中。其余过程在此略去。
总结：
由于没有指定Make目标，那么将使用Makefile.build的默认目标__build，建构的入口点就在此。Make在Makefile.build 和$(obj)/Makefile中寻找 __build依赖的重建规则，主机程序目标还用到了Makefile.host文件。依次变量展开，依赖层层递归重建。


##### 2.指定目标：
  一般情况下，在`(MAKE) $(build)=build_dir  [argv]` 中，通过参数[argv]  指定Make目标时，使用的是`$(obj)/Makefile`文件中构建规则。这时，在`$(obj)/Makefile`文件中不仅要给一些变量赋值，且还包含本目录下目标的重建规则。
 如：
```
%config: scripts_basic outputmakefile FORCE
    $(Q)mkdir -p include/linux include/config
    $(Q)$(MAKE)$(build)=scripts/kconfig $@
```
在此指定Make目标为自动化变量`$@`，当我们输入类似如下的命令：
```
make nitrogen6x_defconfig
```
那么，上述Make命令会被大致替换为：
```
make -f scripts/Makefile.build obj=scripts/kconfig nitrogen6x_defconfig
```
即指定Make的目标为`nitrogen6x_defconfig`。那么在`$(obj)`目录`scripts/kconfig`下的Makefile中包含`nitrogen6x_defconfig`模式匹配规则。其他的流程和上节的"不指定目标"处理类似。在此不再敖述。

#### 三. 入口处：
- 1.顶层Makefile---- 指定目标-----include scripts/kconfig/Makefile

   如在终端中执行配置命令
    ```
    make nitrogen6x_defconfig
    %config: scripts_basic outputmakefile FORCE
    $(Q)mkdir -p include/linux include/config
    $(Q)$(MAKE)$(build)=scripts/kconfig $@
    ```

- 2.auto.conf autoconf.h

    auto.conf.cmd的生成----指定目标-----include scripts/kconfig/Makefile

    ```
    include/config/%.conf:$(KCONFIG_CONFIG)include/config/auto.conf.cmd
    $(Q)$(MAKE)-f$(srctree)/Makefile silentoldconfig
    ```

    将在顶层Makefile中递归到上述1中的`%config`规则，所以，其最终还会包含`scripts/kconfig/Makefile`

- 3.目标编译和链接----不指定目标-----include 各个build目标下的Makefile
    ```
    $(vmlinux-dirs): prepare scripts
        $(Q)$(MAKE) $(build)=$@
    ```

- 4.模块----模块建构中单独讨论
    ```
    $(module-dirs):crmodverdir$(objtree)/Module.symvers
    $(Q)$(MAKE)$(build)=$(patsubst _module_%,%,$@)
    
    modules:$(module-dirs)
    @$(kecho)'  Building modules, stage 2.';
    $(Q)$(MAKE)-f$(srctree)/scripts/Makefile.modpost
    ```

- 5.单目标----不指定目标
    ```
    %.o:%.c prepare scripts FORCE
    $(Q)$(MAKE)$(build)=$(build-dir)$(target-dir)$(notdir $@)
    ```

- 6.子目录递归----不指定目标-----include递归的子目录下Makefile
    ```
    scripts/Makefile.build
    $(subdir-ym):
        $(Q)$(MAKE)$(build)=$@
    ```


#### 四. Makefile.build文件总框架：

```
src:=$(obj)

-include include/config/auto.conf  #if xxx_CONFIG配置选项
include scripts/Kbuild.include    #if_changed等变量

##########################包含obj目录下的Makefile文件############################
kbuild-dir:=$(if $(filter /%,$(src)),$(src),$(srctree)/$(src))
kbuild-file:=$(if $(wildcard $(kbuild-dir)/Kbuild),$(kbuild-dir)/Kbuild,$(kbuild-dir)/Makefile)
include$(kbuild-file)   

############################包含Makefile.lib###################################
include scripts/Makefile.lib  

######################编译主机程序时，要包含Makefile.host文件#####################
ifneq ($(hostprogs-y)$(hostprogs-m),)
include scripts/Makefile.host   
endif

############################定义builtin-target#################################
ifneq ($(strip $(obj-y) $(obj-m) $(obj-n) $(obj-) $(subdir-m) $(lib-target)),)
builtin-target:=$(obj)/built-in.o   ########
endif

############################__build构建规则#################################
__build:$(if $(KBUILD_BUILTIN),$(builtin-target) $(lib-target) $(extra-y))\
    $(if $(KBUILD_MODULES),$(obj-m) $(modorder-target))\
    $(subdir-ym)$(always)
    @:

###################$(obj)/built-in.o,且先要构建依赖$(obj-y)#####################
$(builtin-target):$(obj-y)FORCE   
    $(call if_changed,link_o_target)

############################ 普通模式匹配规则#################################
define rule_cc_o_c
    $(call echo-cmd,checksrc)$(cmd_checksrc)           \
    $(call echo-cmd,cc_o_c)$(cmd_cc_o_c);              \
    $(cmd_modversions)                       \
    $(call echo-cmd,record_mcount)                  \
    $(cmd_record_mcount)                     \
    scripts/basic/fixdep$(depfile)$@ '$(call make-cmd,cc_o_c)' >    \
                                                 $(dot-target).tmp;  \
    rm -f$(depfile);                     \
    mv -f$(dot-target).tmp$(dot-target).cmd
endef

$(obj)/%.o:$(src)/%.c$(recordmcount_source)FORCE  ##$(obj-y)匹配规则
    $(call cmd,force_checksrc)
    $(call if_changed_rule,cc_o_c)
```

## 附录二
本文参考网友博客修改。
### fixdep
#### 简介
新生成一个目标时，会调用`if_changed_dep`检测并更新依赖文件，其中`if_changed_dep`会调用`fixdep`去处理依赖文件，但是将用于生成u-boot的`%.c`文件编译为`%.o`规则除外。在将生成u-boot的`%.c`编译为`%.o`时会调用`rule_cc_o_c`，`rule_cc_o_c`中又会调用`fixdep`生成`%.o`的依赖文件`.%.cmd`。
#### 调用方法
通过`fixdep.c`的`usage()`函数，我们可以看到`fixdep`的用法：
```
fixdep <depfile> <target> <cmdline>

fixdep接收三个参数，分别是：

<depfile>：编译产生的依赖文件*.d
<target>：编译生成的目标
<cmdline>：编译使用的命令
```

编译时，编译器会根据选项-MD自动生成依赖文件\*.d，用fixdep更新\*.d文件生成新的依赖文件.\*.cmd。

fixdep被两个地方调用：

rule_cc_o_c：编译u-boot自身的\*.c文件时，rule_cc_o_c调用fixdep去更新生成%.c的依赖文件.%.cmd

if_changed_dep：适用于除了上述的rule_cc_o_c外的其它目标依赖文件的生成，例如生成主机上执行的程序，处理dts文件等，也包括汇编文件生成\*.o，生成\*.s，\*.lst等。

通过查看fixdep输出的.\*.cmd文件可以知道对应的目标文件\*是如何生成的。

## 附录三
本文参考网友博客修改。
### conf
#### 1. 概述

conf工具的源码位于scripts/kconfig.

#### 2. conf的调用

整个u-boot的配置和编译过程中conf被调用了2次。

##### 2.1 make配置过程调用

u-boot的make配置过程中，完成conf的编译后会随即调用命令：
```mk
scripts/kconfig/conf  --defconfig=arch/../configs/xxx_defconfig Kconfig
```
去生成根目录下的.config文件。

##### 2.2 make编译过程调用

u-boot完成后，执行make命令开始编译时，会检查.config是否比include/config/auto.conf更新，规则如下：
```mk
# If .config is newer than include/config/auto.conf, someone tinkered
# with it and forgot to run make oldconfig.
# if auto.conf.cmd is missing then we are probably in a cleaned tree so
# we execute the config step to be sure to catch updated Kconfig files
include/config/%.conf: $(KCONFIG_CONFIG) include/config/auto.conf.cmd
    $(Q)$(MAKE) -f $(srctree)/Makefile silentoldconfig
    @# If the following part fails, include/config/auto.conf should be
    @# deleted so "make silentoldconfig" will be re-run on the next build.
    $(Q)$(MAKE) -f $(srctree)/scripts/Makefile.autoconf || \
        { rm -f include/config/auto.conf; false; }
    @# include/config.h has been updated after "make silentoldconfig".
    @# We need to touch include/config/auto.conf so it gets newer
    @# than include/config.h.
    @# Otherwise, 'make silentoldconfig' would be invoked twice.
    $(Q)touch include/config/auto.conf
```
实际执行`$(Q)$(MAKE) -f $(srctree)/Makefile silentoldconfig`命令的结果就是检查并更新`scripts/kconfig/conf`文件，然后调用之：
```mk
scripts/kconfig/conf --silentoldconfig Kconfig。
```
为了检查这个命令的输出，可以直接在命令行上执行命令：
```sh
make -f ./Makefile silentoldconfig V=1
make -f ./scripts/Makefile.build obj=scripts/basic
rm -f .tmp_quiet_recordmcount
make -f ./scripts/Makefile.build obj=scripts/kconfig silentoldconfig
mkdir -p include/config include/generated
scripts/kconfig/conf  --silentoldconfig Kconfig
```
其实跟命令行上运行make silentoldconfig是一样的效果。 
执行结果就是：

- 在`include/config`下生成`auto.conf`和`auto.conf.cmd`，以及`tristate.conf`；
- 在`include/generated`下生成`autoconf.h`文件

##### 2.3 调用简述

简而言之：

第一次调用生成.config

scripts/kconfig/conf --defconfig=arch/../configs/rpi_3_32b_defconfig Kconfig

第二次调用生成auto.conf和autoconf.h

scripts/kconfig/conf --silentoldconfig Kconfig

#### 3. conf的源码分析

conf由conf.o和zconf.tab.o链接而来，其中conf.c生成conf.o，是整个应用的主程序；zconf.tab.c生成zconf.tab.o，完成具体的词法和语法分析任务。

##### 3.1 zconf.tab.c

zconf.tab.c用于读取并分析整个Kconfig系统的文件，较为复杂，也比较枯燥，此处略过。

##### 3.2 conf.c

conf.c是conf主程序的文件，通过分析main函数可以大致了解操作流程：

##### 3.2.1 解析参数部分
```c
while ((opt = getopt_long(ac, av, "s", long_opts, NULL)) != -1) {
        if (opt == 's') {
            conf_set_message_callback(NULL);
            continue;
        }
        input_mode = (enum input_mode)opt;
        switch (opt) {
        case silentoldconfig:
            sync_kconfig = 1;
            break;
        case defconfig:
        case savedefconfig:
            defconfig_file = optarg;
            break;
        ...
        }
    }
    if (ac == optind) {
        printf(_("%s: Kconfig file missing\n"), av[0]);
        conf_usage(progname);
        exit(1);
    }
    name = av[optind];
    ...
```
第一次调用 `scripts/kconfig/conf --defconfig=arch/../configs/xxx_defconfig Kconfig`

参数解析后：
```c
input_mode: defconfig_file
defconfig_file: arch/../configs/xxx_defconfig
name = av[optind]: Kconfig
```
第二次调用 `scripts/kconfig/conf --silentoldconfig Kconfig`

参数解析后：
```c
input_mode: silentoldconfig，并设置sync_kconfig为1
name = av[optind]: Kconfig
```
##### 4.2.2 读取Kconfig系统配置文件

设置好input_mode和name后：
```c
conf_parse(name);
    //zconfdump(stdout);
    if (sync_kconfig) {
        name = conf_get_configname();
        if (stat(name, &tmpstat)) {
            fprintf(stderr, _("***\n"
                "*** Configuration file \"%s\" not found!\n"
                "***\n"
                "*** Please run some configurator (e.g. \"make oldconfig\" or\n"
                "*** \"make menuconfig\" or \"make xconfig\").\n"
                "***\n"), name);
            exit(1);
        }
    }
```

调用conf_parse(name)从$(srctree)目录下依次查找名为Kconfig的文件，然后将取得的信息存放到链表中。
如果是silentoldconfig，即sync_kconfig=1，还要调用conf_get_configname()并检查顶层目录下的.config文件是否存在。

##### 4.2.3 读取指定的配置文件
```c
switch (input_mode) {
    case defconfig:
        if (!defconfig_file)
            defconfig_file = conf_get_default_confname();
        if (conf_read(defconfig_file)) {
            printf(_("***\n"
                "*** Can't find default configuration \"%s\"!\n"
                "***\n"), defconfig_file);
            exit(1);
        }
        break;
    case savedefconfig:
    case silentoldconfig:
    case ...:
        conf_read(NULL);
        break;
    ...
    default:
        break;
    }
```
如果是defconfig，调用conf_read(defconfig_file)读取指定的配置文件arch/../configs/rpi_3_32b_defconfig
如果是silentoldconfig，调用conf_read(NULL)读取生成的.config。（conf_read传入的参数为NULL，在conf_read_simple会将读取的文件指向.config）。

##### 4.2.4 检查更新设置
接下来：
```c
if (sync_kconfig) {
        if (conf_get_changed()) {
            name = getenv("KCONFIG_NOSILENTUPDATE");
            if (name && *name) {
                fprintf(stderr,
                    _("\n*** The configuration requires explicit update.\n\n"));
                return 1;
            }
        }
        valid_stdin = tty_stdio;
    } 

    switch (input_mode) {
    case ...:
        break;
    case defconfig:
        conf_set_all_new_symbols(def_default);
        break;
    case savedefconfig:
        break;
    case ...:
    case silentoldconfig:
        /* Update until a loop caused no more changes */
        do {
            conf_cnt = 0;
            check_conf(&rootmenu);
        } while (conf_cnt &&
             (input_mode != listnewconfig &&
              input_mode != olddefconfig));
        break;
    }
```

如果是silentoldconfig，检查.config是否被改动过，并检查各项设置的有效性
如果是defconfig，设置默认值

##### 4.2.5 更新.config，生成相应文件
最后：
```c
if (sync_kconfig) {
        /* silentoldconfig is used during the build so we shall update autoconf.
         * All other commands are only used to generate a config.
         */
        if (conf_get_changed() && conf_write(NULL)) {
            fprintf(stderr, _("\n*** Error during writing of the configuration.\n\n"));
            exit(1);
        }
        if (conf_write_autoconf()) {
            fprintf(stderr, _("\n*** Error during update of the configuration.\n\n"));
            return 1;
        }
    } else if (input_mode == savedefconfig) {
        if (conf_write_defconfig(defconfig_file)) {
            fprintf(stderr, _("n*** Error while saving defconfig to: %s\n\n"),
                defconfig_file);
            return 1;
        }
    } else if (input_mode != listnewconfig) {
        if (conf_write(NULL)) {
            fprintf(stderr, _("\n*** Error during writing of the configuration.\n\n"));
            exit(1);
        }
    }
```
如果是silentoldconfig：

调用conf_get_changed()检查是否更新过，然后调用conf_write(NULL)将更新的项写入到.config文件中
调用conf_write_autoconf()更新以下文件： 
- include/generated/autoconf.h
- include/config/auto.conf.cmd
- include/config/tristate.conf
- include/config/auto.conf

如果是defconfig，进入最后一个(input_mode != listnewconfig)分支，调用conf_write(NULL)，将读取到的所有配置写入.config文件中。

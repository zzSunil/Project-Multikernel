### Overview
目前进跌厂商在k1 kernel tree中提供的rproc driver只支持搭配厂商提供的rtos的启动以及管理，MCS对k1的支持通过移植厂商的rproc driver可以作为通用的rtos管理模块，不用rtos特别的实现rpmsg回调逻辑以及MCS不依赖厂商kernel tree中别的driver，可以单独的作为别的kernel tree的driver编译。

### 环境配置
- 软件配置
	- [OpenEuler 24.03 SP2 image-builder](https://git.oerv.ac.cn/OERV-BSP/image-builder)
	- [uboot firmware](https://github.com/openeuler-riscv/u-boot-build/actions/runs/18710885668)
	- [k1 supported zephyr](https://github.com/zzSunil/zephyr)(个人仓，未上游)
	- [MCS for riscv k1](https://github.com/zzSunil/mcs)(个人仓，未上游)
	- [rvck-olk kernel tree](https://github.com/zzSunil/rvck-olk/tree/k1)(个人仓，未上游)
	- [spacemit openocd tool-chain](https://archive.spacemit.com/tools/openocd/spacemit-openocd-linux-V1.0.1.tar.xz)
	- [spacemit cross-compiler tool-chain](https://archive.spacemit.com/toolchain/spacemit-toolchain-linux-glibc-x86_64-v1.1.2.tar.xz)
- 硬件配置
	- k1 bananapi bpi-f3

### rvck K1 remoteproc 目前状态

由于进跌那边[厂商kernel tree里rproc driver的实现](https://git.oerv.ac.cn/OERV-BSP/kernel-spacemit-k1/src/commit/1b49a8af7eccea4a827536bdc8b7e919091b23ba/drivers/remoteproc/spacemit/k1x-rproc.c)跟esos内部逻辑耦合太多了，而且在rvck-olk中k1的适配缺失mailbox，rproc，pm，pm_doamin，经过尝试后决定根据rproc driver中的实现在mcs中重新实现一次关键步骤(reset/boot control/boot entry)的逻辑。
![](imgs/Pasted%20image%2020251118174338.png)
（之前尝试prot了除了红色的pm以外相关模块到rvck，然后发现esos没法正常初始化😂所以放弃复用他们的driver了）

目前设备树的适配，更新到上方rvck相关仓库链接，mcs/zephyr也进度也同步到上述仓库链接
### K1 specifics

k1由两个有稍微不一样的X60 cluster跟一个Nuclei n308 实时核组成，关于CPU的所有控制寄存器，外设MMIO都可以在[官方文档](https://developer.spacemit.com/documentation?token=ZZrhw4xvHiIVa7kTHlycxrmXn6d#9.3-real-time-cpu)中找到，关于实时核的中断控制器ECLIC跟TIMER的标准跟细节可以在[Nuclei](https://doc.nucleisys.com/nuclei_spec/isa/eclic.html#)跟K1 User Manul中分别找到。
![](imgs/Pasted%20image%2020251128144850.png)
对于在k1上适配MCS，我们只需要做到三件所有rproc driver都需要的做到的事情，一个是为RCPU上运行的RTOS保留一部分内存用来host镜像，rtos运行时堆栈，rpmsg用来在ACPU/RCPU之间通讯，一个是控制RCPU在我们预留出来的内存中加载的rtos镜像入口启动以及RCPU停机，一个是rpmsg底层需要通过平台特殊的mailbox实现核间通讯，所以需要mailbox driver。当然还有rtos侧对于对应cpu平台的BSP支持
#### device tree 适配
关于设备树部分的适配主要分为两部分。一个是mcs相关的，这部分主要是为MCS预留与部分内存用于加载rtos镜像以及为rpmsg需要的vring，endpoint预留一部分内存。![](imgs/Pasted%20image%2020251120182636.png)

另一部分是k1平台相关的，由于26pin GPIO引脚是可以多功能复用的，所以需要在设备树中配置pinctrl寄存器指定外设所使用的GPIO引脚为对应的功能，然后再在mcs的设备树node中引用这个pinctrl。
![](imgs/Pasted%20image%2020251120182933.png)
#### 通过CSR控制RCPU行为
针对MCS kernel module部分的适配主要是通过基于厂商rproc驱动，去掉与esos内部逻辑耦合的rpmsg，power management相关部分。保留控制RCPU相关的步骤，map MMIO，从设备树中拿到reset，通过MMIO映射的控制寄存器配置boot entry，addr map base，startup/shutdown RCPU，assert/de-assert reset。
![](imgs/Pasted%20image%2020251120170459.png)
#### Zephyr n308 BSP支持
目前[zephyr在k1 n308上有基础的BSP支持](https://github.com/zzSunil/zephyr/commit/87179da7de80a1484a1527ad77c508cb1391afe1)，但是由于现在是在进跌时空的rtos/rproc driver先起来初始化外设/引脚/enable clk gate等等之后再通过jtag load_firmware reset init然后直接跳转到指定地址开始运行zephyr，所以在rvck-olk kernel上不依赖厂商的rtos/rproc driver启动zephyr还需要做一些额外的操作。
- 相关外设的clk gate(以uart为例)
	想让uart正常工作需要开启相应时钟的gate，配置pinctrl复用引脚寄存器的值，初始化ns16550 uart的相关寄存器。通过分析，ns16550相关的寄存器zephyr的相关驱动会进行配置，在这里只需要注意由esos(spacemit为k1定制的rtos)ccu驱动完成的对于各种peripheral设备关于始终配置的行为以及pinctrl寄存器关于引脚服用的行为即可。![](imgs/Pasted%20image%2020251120160320.png)
	通过gdb可以看到spacemit自带的rtos对于这一组uart的控制寄存器的配置是0x0007,通过k1的设备树以及esos ccu driver可以看出也就是对应着时钟分频/选择寄存器保持reset后的默认值0，使能reset，pclk/flck gate。这个步骤可以通过在zephyr中直接向MMIO中写入0x0007或者在rvck中设备树中添加r_uart设备以及在pxa_uart驱动中probe这个设备然后通过linux的时钟框架去配置这个控制寄存器完成(因为k1的esos的ccu driver跟他们linux kernel里ccu driver长得几乎一样，他们在esos里跟linux rpmsg的回调函数里都有初始化外设始终的逻辑，感觉至少有一侧是冗余的，毕竟只用做一次就可以了)。
### Debug
- 首先，需要根据自己的 jtag 选型去配置 zephyr 的 openocd 配置文件
	例如这里针对 taigard jtag 修改的 openocd 配置文件![](imgs/Pasted%20image%2020251117124037.png)
	
	使用spacemit官方openocd toolchain也是同理，比如我使用的jtag是tigard，他们工具链里openocd默认的interface里没有相关的配置文件，那就需要在`spacemit-openocd-linux-V1.0.1/share/openocd/scripts/interface`创建一个tigard.cfg把上面patch的这段放到里面然后在启动脚本里把使用interface改成tigard
- 使用zephyr build system编译，烧录，debug(需要先根据zephyr官方教程配置build system与toolchain)
```
$ west build -p always -b bpi_f3/k1/n308 samples/synchronization
$ west flash
$ west debug
```


### 构建指导

#### 准备工作
- 下载[uboot firmware](https://github.com/openeuler-riscv/u-boot-build/actions/runs/18710885668)
使用fastboot刷uboot
```
#!/usr/bin/env bash

set -e

fastboot stage FSBL.bin
fastboot continue
sleep 1
fastboot stage u-boot.itb
fastboot continue
sleep 1
fastboot flash mtd partition-mtd.json
fastboot flash mtd-bootinfo bootinfo_spinor.bin
fastboot flash mtd-fsbl FSBL.bin
fastboot flash mtd-opensbi fw_dynamic.itb
fastboot flash mtd-env u-boot-env-default.bin
fastboot flash mtd-uboot u-boot.itb

echo
read -p "Finished. Press ENTER to exit. "

```

#### 一键构建image
- 下载[image-builder](https://git.oerv.ac.cn/zzSunil/image-builder/src/branch/master)
```bash
$ make openEuler-24.03-LTS-SP1-base-SpacemiT-K1-extlinux_mcs
```
- 将`build/dist/openEuler/24.03-LTS-SP1/SpacemiT-K1`中生成的image烧入sd卡(第一次启动会有些慢，因为需要编译dkms，可能需要等大概2分钟)
#### 下载软件包
- [mcs kernel module的dkms包，mcs用户态工具，rtos配置文件，rtos镜像 repo](https://build-repo.tarsier-infra.isrc.ac.cn/home:/zz-_:/branches:/openEuler:/24.03:/SP2:/Epol/standard_riscv64/)
- [支持mcs的rvck-olk kernel repo](https://build-repo.tarsier-infra.isrc.ac.cn/home:/zz-_:/branches:/openEuler:/24.03:/SP3:/Everything/standard_riscv64/)

**需要把extlinux.conf中append的cmdline参数中的console改成`console=ttySP0,115200`**
#### 手动编译
- 准备工做：
	1. 下载[github action uboot固件](github.com/openeuler-riscv/u-boot-build/actions/runs/18710885668)
	2. 使用fastboot刷写uboot
	3. 准备sd卡，使用[image builder](https://git.oerv.ac.cn/OERV-BSP/image-builder)生成openEuler-24.03-LTS-SP2-base-SpacemiT-K1-extlinux镜像,并烧写在sd卡上

1. 安装 libmetal 的依赖 libsysfs, kernel module编译需要的headers
```bash
$ dnf install sysfsutils sysfsutils-devel kernel-devel kernel-headers
```

2. 编译,安装 libmetal
```bash
$ git clone https://github.com/OpenAMP/libmetal.git
$ mkdir libmetal/build && cd libmetal/build
$ cmake -DCMAKE_INSTALL_LIBDIR=/usr/lib64 .. & make install -j
```

3. 编译,安装 libopenamp
```bash
$ git clone https://github.com/OpenAMP/open-amp.git
$ mkdir open-amp/build && cd open-amp/build
$ cmake -DCMAKE_INSTALL_LIBDIR=/usr/lib64 .. & make install -j
```

4. 编译 mcs_km, mica
```Bash
$ git clone git@github.com:zzSunil/mcs.git
$ mkdir mcs/build && cd mcs/build
$ cmake .. && make -j
$ cd $PATH_TO_RVCK #在rvck-olk kernel tree下cross-compile mcs kernel module
$ make -j31 ARCH=riscv CROSS_COMPILE=/PATH_TO/spacemit-toolchain-linux-glibc-x86_64-v1.1.2/bin/riscv64-unknown-linux-gnu- M=/PATH_TO/mcs/mcs_km TARGET=K1
```

5. 安装mica，micad，加载mcs_km.ko
```bash
$ cp mica/micactl/mica.py /usr/bin/mica
$ cp build/mica/micad/micad /usr/bin
$ insmod mcs_km/mcs_km.ko
```

- 由于需要为client OS预留内存，同时kernel module也会识别设备树中的信息进行初始化，需要为qemu准备适配mcs的设备树文件。k1的设备树patch放在MCS repo下tools

### 测试步骤

分别连接RCPU，ACPU UART
![](imgs/Pasted%20image%2020251218173914.png)

进入shell之后
1. 加载kernel module
```
$ modprobe mcs_km  
```
2. 启动 micad 服务
```
$ micad
```
3. 通过 mica 创建 rtos 实例
```
$ mica create /lib/firmware/mcs/k1-zephyr.conf
```
4. 启动/停止 rtos
```
$ mica start k1-zephy
$ mica stop k1-zephyr
```

预期行为
ACPU uart输出linux kernel log
RCPU uart输出zephyr的log
![](imgs/Pasted%20image%2020251218175404.png)
### Links & Docs
- [k1 CPU System Doc](https://developer.spacemit.com/documentation?token=ZZrhw4xvHiIVa7kTHlycxrmXn6d)(所有驱动相关的控制寄存器/MMIO寄存器都可以在这里找到)
- [k1 RCPU System Do](https://doc.nucleisys.com/nuclei_spec/isa/eclic.html#)(实时核相关手册)
- [bananapi Bpi-f3 Doc](https://docs.banana-pi.org/zh/BPI-F3/BananaPi_BPI-F3)(可以看到gpio26pin引脚图，方便接线)


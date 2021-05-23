# Rocket Chip在ZYNQ上的实现

该仓库包含在Vivado 2016.2上的各种Zynq FPGA开发板（Zybo，Zedboard，ZC706，PYNQ-z2）上运行RISC-V rocket chip所需的文件。 （注：因为Vivado版本问题，推荐使用Ubuntu16.04操作系统）

### 如何使用该README：

该README主要包含3部分：

[1 - 快速开始](#quickinst)：使用编译好的文件极速上手，无需下载各种工具。

[2 - RISCV工具链（riscv-tools）编译](#toolchain)：安装编译rocket chip所需的工具链。

[3 - 工程编译的详细步骤](#compile)：从头开始一步步编译整个工程。

[附录](#appendices)：主机和开发板传输文件的方法。

注：以下的`$REPO`均代表`fpga-pynq`repo在本地的目录，建议执行以下命令将REPO加入环境变量（替换repo在本地的目录）：

```
$ export REPO=repo在本地的目录
```

## 目录

+ [0 - 克隆整个工程到本地](#start)
+ [1 - 快速开始](#quickinst)
+ [2 - RISCV工具链（riscv-tools）编译](#toolchain)
+ [3 - 工程编译的详细步骤](#compile)
  + [创建工程](#setup)
  + [生成比特流文件](#bitstream)
  + [编译FSBL](#fsbl)
  + [编译u-boot](#u-boot)
  + [Building u-boot for the Zynq ARM Core](#u-boot)
  + [创建boot.bin](#boot.bin)
  + [编译zynq ARM的linux内核](#arm-linux)
  + [生成设备树文件](#arm-dtb)
  + [启动](#booting)
+ [附录](#appendices)
+ [说明](#note)

## 0)<a name="start"></a> 克隆整个工程到本地

为了方便大家快速获取源码，已将全部源码（包括子模块）打包上传到百度网盘，可以直接下载。

```
链接：https://pan.baidu.com/s/1mTCcKG0EiFdxq4C5HTey3w 
提取码：1234 
```

```
$ cat fpga-pynq.0* > fpga-pynq.tar.gz     #组装文件
$ md5sum  fpga-pynq.tar.gz > md5   #计算MD5校验码
$ cmp md5 md5sum    #比对校验码，如果此处没有任何输出，则为正确
$ tar -zxvf fpga-pynq.tar.gz    #解压文件
```

或者从github获取源码：

```
$ git clone https://github.com/huozf123/fpga-pynq.git
$ cd fpga-pynq
$ git submodule update --init --recursive    #快速开始不需要执行该指令，自己编译工程才需要
```

1)<a name="quickinst"></a> 快速开始
------------------

*用预先编译好的镜像，运行hello world程序在rocket chip上 (注：此环节无需安装任何工具)*

首先，将SD卡通过读卡器插入系统，进入目标开发板的目录 (可选项是 `zybo`, `zedboard`,  `zc706`，`pynq-z2`)。执行如下命令将镜像拷入SD卡（或者手动将`$REPO/pynq-z2/fpga-images-pynq`目录下的四个文件拷贝至SD卡），将“SD卡的路径”替换为实际SD卡的路径：

    $ make load-sd SD=SD卡的路径

最后，弹出SD卡，将其插入开发板，将开发板的启动跳线设置为“ SD”，然后打开开发板的电源。 使用网线连接至开发板，打开主机终端用SSH登录ARM端的linux系统（用户名密码均为*root*），并在rocket chip上运行hello world： 

    $ ssh root@192.168.1.5
    root@zynq:~# ./fesvr-zynq pk hello
    hello!

## 2)<a name="toolchain"></a> RISCV工具链（riscv-tools）编译

（如果机器上已经有编译好的工具链，则只需将其加入环境变量即可，这一步可以跳过）

1）安装依赖：

```
$ sudo apt-get install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev libusb-1.0-0-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev device-tree-compiler pkg-config libexpat-dev
```

2）编译工具链，此处需要指定工具链要安装的目标路径（绝对路径）：

```
$ export RISCV=工具链要安装的目的路径
$ export PATH=${RISCV}/bin:$PATH
$ cd $REPO/rocket-chip/riscv-tools/
$ ./build.sh
```

注：此处需要GCC>=4.8，详情请参照[README](https://github.com/riscv/riscv-tools#readme)。

3)<a name="compile"></a> 工程编译的详细步骤
-------------------------

*前提：安装好的Vivado 2016.2 ，一个可以运行scala代码的JVM（注：测试使用的java版本为1.8.0_271，如果编译rocket chip过程中遇到java错误，可能是java版本的原因）*

首先添加Vivado相关的环境变量，执行（替换掉“你的vivado安装目录”）：

```
$ source 你的vivado安装目录/Vivado/2016.2/settings64.sh
$ source 你的vivado安装目录/SDK/2016.2/settings64.sh 
```

因为Vivado、SDK存在bug，所以需要执行以下命令（替换“你的vivado安装目录”）：

```
$ sudo apt-get install libgoogle-perftools-dev
$ export SWT_GTK3=0
$ sed -i "11,15s/^/#/" 你的vivado安装目录/Vivado/2016.2/.settings64-Vivado.sh    #注释该文件第11-15行
```

然后初始化子模块，进入目标开发板的目录，执行：

    $ make init-submodules

### 3.1) <a name="setup"></a> 创建工程

首先，进入目标开发板的目录，执行如下命令生成工程。 

	$ make project

### 3.2) <a name="bitstream"></a> 生成比特流文件

然后，我们通过如下命令打开vivado:

	$ make vivado

然后点击左下角的*Generate Bitstream*按钮， Vivado将自动生成比特流文件。该文件位置为：

`$REPO/pynq-z2/pynq_rocketchip_ZynqFPGAConfig/pynq_rocketchip_ZynqFPGAConfig.runs/impl_1/rocketchip_wrapper.bit`

下一步，点击*File -> Export -> Export Hardware*。这将创建以下目录：

`$REPO/pynq-z2/pynq_rocketchip_ZynqFPGAConfig/pynq_rocketchip_ZynqFPGAConfig.sdk`

该目录包含的各种文件向SDK提供有关硬件的信息。


### 3.3) <a name="fsbl"></a> 编译FSBL

在Vivado界面点击*File -> Launch SDK* 打开SDK：

1) 点击 *File -> New -> Application Project*

2) 在弹出的新窗口中，输入"FSBL" 作为Project name，其他项保持默认：

3) 点击*Next*，然后依次点击*Zynq FSBL* 和*Finish*。然后SDK将继续自动编译FSBL。

4) 编译完成后，继续下一步。

### 3.4) <a name="u-boot"></a> 编译u-boot

打开一个新的终端，进入目标开发板的目录，执行如下命令：

	$ source 你的vivado安装目录/Vivado/2016.2/settings64.sh
	$ source 你的vivado安装目录/SDK/2016.2/settings64.sh 
	$ make arm-uboot

编译好的u-boot所在位置为：`$REPO/pynq-z2/soft_build/u-boot.elf`。

### 3.5) <a name="boot.bin"></a> 创建boot.bin

回到SDK界面，点击 *Xilinx Tools -> Create Zynq Boot Image*。

1) 点击*Output BIF file path*后面的*Browse..*,然后找到并选择`$REPO/pynq-z2/deliver_output`。

2) 点击右下角的*Add*，并在弹出的对话框中点击*Browse*，找到如下文件（First Stage BootLoader）：

`$REPO/pynq-z2/pynq_rocketchip_ZynqFPGAConfig/pynq_rocketchip_ZynqFPGAConfig.sdk/FSBL/Debug/FSBL.elf`

*Partition type*选择bootloader，然后点击*OK*。

3) 再一次点击 *Add*，并在弹出的对话框中点击*Browse*，找到如下文件（bitstream）：

`$REPO/pynq-z2/pynq_rocketchip_ZynqFPGAConfig/pynq_rocketchip_ZynqFPGAConfig.runs/impl_1/rocketchip_wrapper.bit`

*Partition type* 选择datafile，然后点击*OK*。

4)  再一次点击 *Add*，并在弹出的对话框中点击*Browse*，找到如下文件（uboot）：

`$REPO/pynq-z2/soft_build/u-boot.elf`

*Partition type* 选择datafile，然后点击*OK*。

5) 点*Create Image*。这将产生 `BOOT.bin` 文件在 `$REPO/pynq-z2/deliver_output` 目录下。

进行完以上5个步骤之后，如果再次修改其中的文件，可以进入目标开发板的目录，通过如下命令快速生成boot.bin文件（注：最终写入到SD卡中的boot.bin文件名不区分大小写）。

    $ make deliver_output/boot.bin

### 3.6) <a name="arm-linux"></a> 编译zynq ARM的linux内核

进入目标开发板的目录，然后编译linux内核：

	$ make arm-linux

### 3.7) <a name="arm-dtb"></a> 生成设备树文件

生成linux的dtb：

	$ make arm-dtb

### 3.8) <a name="booting"></a> 启动

此时，`$REPO/pynq-z2/deliver_output` 目录下包含如下文件：

* `BOOT.bin` - 包含FSBL、bitstream、u-boot。
* `uImage` - zynq ARM端的Linux内核。
* `devicetree.dtb` - Linux需要的设备树文件。
* `uramdisk.image.gz` - ARM linux的文件系统。

最终只需将linux根文件系统复制到该目录下即可完成SD卡内所有文件的准备工作，进入目标开发板的目录，执行：

    $ cp fpga-images-pynq/uramdisk.image.gz ./deliver_output/

现在将`deliver_output/`中的如下四个文件拷贝到SD卡中，然后将SD卡插入到pynq-z2开发板中，将开发板右上角跳线帽调整到SD端（即从SD卡启动）。SD卡内的目录结构如下：

	SD_ROOT/
	|-> boot.bin
	|-> devicetree.dtb
	|-> uImage
	|-> uramdisk.image.gz

此时已经完成了所有工作，打开开发板电源，使用网线（用户名密码均为*root*）连接至开发板并运行hello world： 

    $ ssh root@192.168.1.5
    root@zynq:~# ./fesvr-zynq pk hello
    hello!

<a name="appendices"></a> 附录
------------


### 主机与开发板传输文件

#### 通过以太网传输文件
最简单的方法，使用scp在线传输文件：

    $ scp file root@192.168.1.5:~/

*注意*：上电期间对文件系统的修改不会写入`uramdisk.image.gz`文件中，如需永久修改文件系统参见如下：

#### 修改linux文件系统
1）首先需要安装uboot tools：

```
sudo apt-get install u-boot-tools -y
```

2）将SD卡通过读卡器插入主机。

3）进入目标开发板的目录，解压文件系统：

    $ cd $REPO/pynq-z2
    $ make ramdisk-open

解压后的文件系统位置为：`$REPO/pynq-z2/ramdisk`，可以在此处修改文件系统。

4）修改完成后压缩文件系统，覆盖旧的文件系统：

    $ make ramdisk-close

（注：文件系统的位置为：`$REPO/pynq-z2/fpga-images-pynq/uramdisk.image.gz`）

5）将文件系统`$REPO/pynq-z2/fpga-images-pynq/uramdisk.image.gz`拷贝到SD卡。

## <a name="note"></a> 说明

此工程基于[fpga-zynq](https://github.com/ucb-bar/fpga-zynq)，如果需要更加深入的了解，请参考[fpga-zynq](https://github.com/ucb-bar/fpga-zynq)。

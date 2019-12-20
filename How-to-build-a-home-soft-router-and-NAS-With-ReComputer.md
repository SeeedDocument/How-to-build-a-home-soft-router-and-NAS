# 如何构建一台家庭软路由和NAS



## 1. 硬件准备

- ReComputer电脑主板，本教程中使用的是8GB RAM + 64GB eMMC的版本
- SSD，要构建NAS，板载的64G显然是不够用的，我们需要大量的硬盘，推荐使用SSD，因为相比于HDD，SSD的寿命长很多，不需要考虑RAID，我们可以选择的SSD有m.2 SATA SSD或者m.2 NVME SSD或者2.5'' SATA SSD
- 一个8CM 4PIN散热风扇，在火热的夏天，靠散热鳍片自然散热是不够的，我们想要静音，因此选择了较大的风扇，笔者想要一个更轻薄的机壳，因此放弃了9015风扇而选择了8010风扇
- 一个机壳，别担心，本教程将公开设计文件，让您也可以仿制一个拥有简单散热风道的紧凑机壳
- Grove - OLED Display 0.96"，ReComputer主板上有一颗Arduino微控制器，不要放过耍酷的好机会
- 一些线材，比如SATA数据线、SSD供电线、风扇供电线、Grove连接线等，有些线材需要动手改造，不过很简单

 ![](/Users/Jack/Documents/Seeed/ReComputer/10.jpg)

 ![](/Users/Jack/Documents/Seeed/ReComputer/11.jpg)

 ![](/Users/Jack/Documents/Seeed/ReComputer/12.jpg)



**机壳**

鉴于设计尺寸，在选用亚克力板材时，请选用厚度小于3mm的板材。设计文件见附件ReComputer_DarkBox.dxf。

立柱请选用25mm x 4和27mm x 4。



**线材**

在制作线材时，请留意HDD_PWR接口的针脚定义。散热风扇可以从HDD_PWR接口中取用12V电源。

![](/Users/Jack/Documents/Seeed/ReComputer/13.jpg)



## 2. 组装

第1步，先将Grove - OLED Display 0.96"安装到机壳上

![](/Users/Jack/Documents/Seeed/ReComputer/20.jpg)

![](/Users/Jack/Documents/Seeed/ReComputer/21.jpg)



第2步，将散热风扇安装到机壳上

![](/Users/Jack/Documents/Seeed/ReComputer/22.jpg)



第3步，将2.5‘’ SSD安装到机壳上

![](/Users/Jack/Documents/Seeed/ReComputer/23.jpg)

![](/Users/Jack/Documents/Seeed/ReComputer/24.jpg)



第4步，接线

4PIN散热风扇的接口如下图所示，它有转速检测脚TACH和转速控制脚PWM，我们分别将它们连接到Arduino的12和13脚（这取决于Arduino程序中的定义）。

![](/Users/Jack/Documents/Seeed/ReComputer/25fan_pinout.png)

![](/Users/Jack/Documents/Seeed/ReComputer/25.jpg)



将Grove - OLED Display 0.96"连接到I2C，同时接通电源和地线。

![](/Users/Jack/Documents/Seeed/ReComputer/26.jpg)



将SAMD21的串口与Intel CPU的串口相连。

![](/Users/Jack/Documents/Seeed/ReComputer/25uart.jpg)



第5步，将机壳侧面与前后盖板组装，并旋紧螺钉。

![](/Users/Jack/Documents/Seeed/ReComputer/27.jpg)

![](/Users/Jack/Documents/Seeed/ReComputer/28.jpg)



**散热风道**

风扇吸入的冷风依次流过CPU散热鳍片和SSD硬盘，有效地为大容量SSD散热。

![](/Users/Jack/Documents/Seeed/ReComputer/29.jpg)



## 3. Proxmox VE虚拟环境安装

我们需要一个不小于8G的U盘以制作安装媒体，进入<https://www.proxmox.com/en/downloads>下载最新的Proxmox VE ISO镜像。

使用Etcher写入U盘。

将ReComputer连接键盘、鼠标和显示器，将U盘插入ReComputer启动，连续点按“F7”键进入启动媒体选择界面，选择U盘启动。

PVE的安装过程非常简单，但有一点非常重要的需要提醒：

**PVE不能安装到eMMC中。**

这是因为PVE团队认为eMMC不如SSD寿命长，他们不允许PVE安装在eMMC媒体上。

![](/Users/Jack/Documents/Seeed/ReComputer/pve-grub-menu.png)

（图片下载自PVE官网，笔者安装的是6.0版）

安装过程如果遇到问题，请参阅PVE安装文档<https://pve.proxmox.com/wiki/Installation>。

ReComputer有2个网口，任意选择一个作为PVE的管理网口（另外一个网口将用作软路由系统的WAN口）。



## 4. Arduino程序

我们将使用ReComputer板载的SAMD21（Seeeduino Cortex-M0+兼容）控制风扇转速，根据CPU温度动态调整风扇转速。同时，会将PVE系统的一些信息显示在OLED显示屏上。

程序的思路是这样的：

- PVE是一个Debian Linux Box，我们可以很灵活地编写一段程序，读取CPU的温度
- SAMD21的USB串口已经连接到了Intel CPU的USB接口上，我们可以通过这个USB接口烧写程序
- SAMD21的另一个串口Serial1连接到了Intel CPU的串口上，我们可以通过这个串口相互通信（相比于USB串口，笔者认为硬串口更加稳定可靠）
- 编写一个简单的Arduino程序，从Serial1读取CPU温度，PID控制风扇转速，驱动刷新OLED显示屏

非常简单吧，好了，Arduino程序的源代码在这里

<https://github.com/KillingJacky/DarkBox>

### 4.1 编译

我们要做的是用Arduino IDE打开这个程序，选择Seeeduino Cortex-M0+编译，然后通过编译日志找到bin文件。

![image-20191112210126228](/Users/Jack/Documents/Seeed/ReComputer/401.png)



![image-20191112210342437](/Users/Jack/Documents/Seeed/ReComputer/402.png)



### 4.2 烧写

将Arduino IDE编译生成的ReComputer.ino.bin使用scp命令拷贝到PVE中，像这样

```
scp ReComputer.ino.bin root@192.168.1.x:~
```

ssh进入PVE系统

```
ssh root@192.168.1.x
```

下载烧写工具bosaac

```
wget http://downloads.arduino.cc/tools/bossac-1.7.0-x86_64-linux-gnu.tar.gz
tar zxvf bossac-1.7.0-x86_64-linux-gnu.tar.gz
cp bossac-1.7.0/bossac /usr/bin/
chmod a+x /usr/bin/bossac
```

令Arduino进入bootloader模式，需要短接Reset与Gnd两次

![image-20191113230804316](/Users/Jack/Documents/Seeed/ReComputer/resetArduino.png)

通过bossac工具烧写Arduino程序

```
bossac -i -d --port=/dev/ttyACM0 -U true -e -w -v ReComputer.ino.bin -R
```

此时，您将看到OLED显示屏上出现这样的画面

![](/Users/Jack/Documents/Seeed/ReComputer/oled_gui.jpg)



它显示了CPU的温度和风扇的转速，并且当CPU温度低于45度时，风扇会停转。

同时显示了系统的负载历史和当前的内存使用量。

当然，别忘了安装PVE中的脚本，也请参阅github仓库中的README。

好了，到这里我们完成了所有硬件工作，我们有了一个可以智能散热的轻薄PVE服务器，它有2T硬盘空间可以让我们安装多个虚拟机，同时也可以做为NAS的存储媒体。



## 5. 安装软路由系统

ReComputer主板拥有两个千兆网口，这使它能够很容易地构建软路由系统，软路由系统有着比普通路由器较强大的功能，可以给您一个更专业的家庭网络环境。

笔者选择了使用难度不高、社区活跃的**lede(OpenWrt)**系统。

我们来构思一下网络拓扑图。

![image-20191116233322566](/Users/Jack/Documents/Seeed/ReComputer/500networkArch.png)

第1步，下载安装镜像

https://drive.google.com/file/d/1-R5mJOu43bKWHv8ViK2V1dtE4zBLDYyU/view?usp=sharing

这是笔者从lede第三方修改版源码编译而来一个镜像。



第2步，上传磁盘镜像到PVE

```
scp /PATH/TO/openwrt-x86-64-combined-squashfs.qcow2 root@192.168.32.222:~
```

笔者编译镜像时直接输出了qcow2格式，如果您下载的镜像为img格式，可以使用下面的命令进行格式转换。

```
qemu-img convert -f raw -O qcow2 lede-xxxxxxx-combined-ext4.img vm-100-disk-1.qcow2
```



第3步，创建虚拟机，并导入磁盘镜像。

创建WAN网口，重启PVE以使添加的WAN可用。

![image-20191117161646454](/Users/Jack/Documents/Seeed/ReComputer/503createWanBridge.png)

![image-20191117164131776](/Users/Jack/Documents/Seeed/ReComputer/503wanActive.png)



创建虚拟机，最后的配置如下图（创建虚拟机Wizard走完后，手动添加第2块网卡，删除硬盘）

![image-20191117161819910](/Users/Jack/Documents/Seeed/ReComputer/504ledeSummary.png)

接下来我们导入lede的磁盘镜像，

```
root@pve-home:~# qemu-img check openwrt-x86-64-combined-squashfs.qcow2
No errors were found on the image.
685/2824 = 24.26% allocated, 0.00% fragmented, 0.00% compressed clusters
Image end offset: 45219840
root@pve-home:~# qemu-img info openwrt-x86-64-combined-squashfs.qcow2
image: openwrt-x86-64-combined-squashfs.qcow2
file format: qcow2
virtual size: 177M (185073664 bytes)
disk size: 43M
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
root@pve-home:~# qm importdisk 100 openwrt-x86-64-combined-squashfs.qcow2 local-lvm
  Rounding up size to full physical extent 180.00 MiB
  Logical volume "vm-100-disk-0" created.
    (100.00/100%)
```

注意：100为创建的虚拟机编号，请根据您的实际情况修改为相应值。

此时，我们可以在local-lvm下看到我们刚刚导入的磁盘。

![image-20191117163326117](/Users/Jack/Documents/Seeed/ReComputer/505diskImported.png)

同时，在虚拟机的硬件列表下也可以看到我们导入的磁盘。

![image-20191117163523743](/Users/Jack/Documents/Seeed/ReComputer/506diskImported2.png)

双击它，将其添加

![image-20191117163625885](/Users/Jack/Documents/Seeed/ReComputer/507addDisk.png)

最后的硬件列表像这样：

![image-20191117163718793](/Users/Jack/Documents/Seeed/ReComputer/508finalHardwareSummary.png)

启动虚拟机，打开Console查看kernel log，当打印`random: crng init done`时，敲击回车，看到shell，启动成功。

![image-20191117164609593](/Users/Jack/Documents/Seeed/ReComputer/509ledeBootup.png)

此时，lede的内网IP为192.168.1.1，想要访问此IP，我们需要用电脑直连ReComputer的LAN网口，并将电脑的IP地址配置为静态IP 192.168.1.x。

![image-20191117165532300](/Users/Jack/Documents/Seeed/ReComputer/510configLaptopNetwork.png)

在浏览器中输入地址192.168.1.1，出现OpenWrt的登陆界面，默认用户名root，默认密码password。

![image-20191117165632253](/Users/Jack/Library/Application Support/typora-user-images/image-20191117165632253.png)

接下来如何玩耍OpenWrt已不在本文叙述范围内，just study and enjoy！



## 6. 安装NAS系统

NAS是家庭网络中越来越重要的服务之一，通过PVE虚拟环境我们可以轻松地安装NAS系统，这里我们选择了开源NAS系统openmediavault。



第1步，下载安装镜像

https://sourceforge.net/projects/openmediavault/files/5.0.5/openmediavault_5.0.5-amd64.iso/download

第2步，上传安装镜像到PVE

![image-20191114152513579](/Users/Jack/Documents/Seeed/ReComputer/602uploadInstaller.png)



第3步，创建PVE虚拟机，最后的配置如下：

![image-20191117110324189](/Users/Jack/Documents/Seeed/ReComputer/603omvConfig.png)



第4步，启动刚刚创建的虚拟机，安装openmediavault，一路确认即可。

![image-20191117110717036](/Users/Jack/Documents/Seeed/ReComputer/604installOMV.png)

![image-20191117111323934](/Users/Jack/Documents/Seeed/ReComputer/605installOMVDone.png)

安装完后会出现上图界面，此时需要移除虚拟机的CD-ROM中的ISO镜像。

![image-20191117111506366](/Users/Jack/Documents/Seeed/ReComputer/606removeCDROM.png)

移除后回到“Console”，按下回车键使虚拟机重启。

![image-20191117111854853](/Users/Jack/Documents/Seeed/ReComputer/607omvFirstBoot.png)

根据屏幕上显示的IP地址，在浏览器中进入此IP地址的网页。然后用默认的用户名和密码登陆，默认用户名admin，默认密码openmediavault。

![image-20191117112155601](/Users/Jack/Documents/Seeed/ReComputer/608loginOMV.png)

![image-20191117112400979](/Users/Jack/Documents/Seeed/ReComputer/609omvWebUIFirstView.png)

到这里基本的openmediavault系统已经安装完成了，接下来我们将要把SSD硬盘直通进来，以提高OMV系统对硬盘的读写效率。



第5步，硬盘直通。

根据PVE的文档，我们需要先使能IOMMU。ssh进PVE后，执行：

```
root@pve-home:~# vim /etc/default/grub
```

在`GRUB_CMDLINE_LINUX_DEFAULT`后添加`intel_iommu=on`,添加完后是这样

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
```

运行`update-grub`

```
root@pve-home:~# update-grub
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.0.15-1-pve
Found initrd image: /boot/initrd.img-5.0.15-1-pve
Found memtest86+ image: /boot/memtest86+.bin
Found memtest86+ multiboot image: /boot/memtest86+_multiboot.bin
Adding boot menu entry for EFI firmware configuration
done
```

You have to make sure the following modules are loaded. This can be achieved by adding them to ‘*/etc/modules*’

```
 vfio
 vfio_iommu_type1
 vfio_pci
 vfio_virqfd
```

After changing anything modules related, you need to refresh your `initramfs`. On Proxmox VE this can be done by executing:

```
root@pve-home:~# update-initramfs -u -k all
```

Finally reboot to bring the changes into effect and check that it is indeed enabled.

```
root@pve-home:~# dmesg | grep -e DMAR -e IOMMU -e AMD-Vi
...
[    1.810500] DMAR: Setting RMRR:
[    1.810644] DMAR: Setting identity map for device 0000:00:02.0 [0x77800000 - 0x7fffffff]
[    1.810794] DMAR: Setting identity map for device 0000:00:15.0 [0x75935000 - 0x75954fff]
[    1.810805] DMAR: Prepare 0-16MiB unity mapping for LPC
[    1.810891] DMAR: Setting identity map for device 0000:00:1f.0 [0x0 - 0xffffff]
[    1.810959] DMAR: Intel(R) Virtualization Technology for Directed I/O
```

If you see the outputs above, IOMMU is enabled.

查看我们要直通的硬盘在哪个PCI接口上，SATA3接口所连接的SATA控制器在00:12.0接口上。

```
root@pve-home:~# lspci -nn
00:00.0 Host bridge [0600]: Intel Corporation Device [8086:31f0] (rev 03)
00:02.0 VGA compatible controller [0300]: Intel Corporation Device [8086:3185] (rev 03)
00:0c.0 Network controller [0280]: Intel Corporation Device [8086:31dc] (rev 03)
00:0e.0 Audio device [0403]: Intel Corporation Device [8086:3198] (rev 03)
00:0f.0 Communication controller [0780]: Intel Corporation Celeron/Pentium Silver Processor Trusted Execution Engine Interface [8086:319a] (rev 03)
00:12.0 SATA controller [0106]: Intel Corporation Device [8086:31e3] (rev 03)
00:13.0 PCI bridge [0604]: Intel Corporation Device [8086:31d8] (rev f3)
00:14.0 PCI bridge [0604]: Intel Corporation Device [8086:31d6] (rev f3)
00:14.1 PCI bridge [0604]: Intel Corporation Device [8086:31d7] (rev f3)
00:15.0 USB controller [0c03]: Intel Corporation Device [8086:31a8] (rev 03)
00:17.0 Signal processing controller [1180]: Intel Corporation Device [8086:31b4] (rev 03)
00:17.1 Signal processing controller [1180]: Intel Corporation Device [8086:31b6] (rev 03)
00:17.2 Signal processing controller [1180]: Intel Corporation Device [8086:31b8] (rev 03)
00:18.0 Signal processing controller [1180]: Intel Corporation Celeron/Pentium Silver Processor Serial IO UART Host Controller [8086:31bc] (rev 03)
00:18.1 Signal processing controller [1180]: Intel Corporation Celeron/Pentium Silver Processor Serial IO UART Host Controller [8086:31be] (rev 03)
00:18.2 Signal processing controller [1180]: Intel Corporation Celeron/Pentium Silver Processor Serial IO UART Host Controller [8086:31c0] (rev 03)
00:18.3 Signal processing controller [1180]: Intel Corporation Celeron/Pentium Silver Processor Serial IO UART Host Controller [8086:31ee] (rev 03)
00:19.0 Signal processing controller [1180]: Intel Corporation Celeron/Pentium Silver Processor Serial IO SPI Host Controller [8086:31c2] (rev 03)
00:1c.0 SD Host controller [0805]: Intel Corporation Celeron/Pentium Silver Processor SDA Standard Compliant SD Host Controller [8086:31cc] (rev 03)
00:1e.0 SD Host controller [0805]: Intel Corporation Device [8086:31d0] (rev 03)
00:1f.0 ISA bridge [0601]: Intel Corporation Device [8086:31e8] (rev 03)
00:1f.1 SMBus [0c05]: Intel Corporation Celeron/Pentium Silver Processor Gaussian Mixture Model [8086:31d4] (rev 03)
01:00.0 Non-Volatile memory controller [0108]: Samsung Electronics Co Ltd NVMe SSD Controller SM961/PM961 [144d:a804]
02:00.0 Ethernet controller [0200]: Intel Corporation I211 Gigabit Network Connection [8086:1539] (rev 03)
03:00.0 Ethernet controller [0200]: Intel Corporation I211 Gigabit Network Connection [8086:1539] (rev 03)
```



接下来我们回到PVE的Web GUI，在OMV虚拟机下选择添加新硬件

![image-20191117114829217](/Users/Jack/Documents/Seeed/ReComputer/610pciPassthrough.png)

![image-20191117155102090](/Users/Jack/Documents/Seeed/ReComputer/611selectPCI.png)

添加后，关闭OMV虚拟机，再重新启动，即可发现OMV中识别了我们直通的硬盘。

![image-20191117155433087](/Users/Jack/Documents/Seeed/ReComputer/612seeTheNewDisk.png)

接下来就参照openmediavault的文档enjoy吧。










# Windows平板（X86架构）安装Ubuntu

*<!--参考我自己的垃圾平板，闲鱼上找的，芯片Z3735G，1G+15G-->*

## 使用工具：

+ rufus-3.21.exe刻录iso镜像到U盘
+ bootia32.efi文件
+ micro-usb otg转换线
+ usb hub扩展线，需要键盘和鼠标能同时操作
+ 一个大于4G的U盘

## 步骤

1. 到ubuntu官网https://ubuntu.com/下载ubuntu18的iso镜像，需要amd64的。18的内存要求为1G刚好和我的平板一样。ubuntu16发现没有网卡驱动，而20又太卡，安装时不能输入用户名密码这些卡住。

2. 使用rufus-3.21.exe将iso镜像刻录。设置：分区类型为GPT，目标系统类型为UEFI(非CSM)模式，文件系统为NTFS，簇大小默认，快速格式化（否则，格式化非常慢）。完毕后，将bootia32.efi文件拷贝到U盘的EFI/boot/目录下。

3. otg转换线连接到平板，连接hub，插上键盘，插入U盘。平板长按电源开机，出现画面时快速按Del/Esc，我的是Esc。进入到设置界面，**找到boot->secure boot menu -> secure boot，将其设置为disabled**(否则无法识别U盘，从U盘启动安装)。以及**fast boot设置为disabled。不同的bios可能位置不同。**然后F4保存设置，自动进入到EFI shell。执行命令exit退出，就来到grub引导的界面（install ubuntu）。

4. 在grub引导界面中，选择install ubuntu，进入到U盘的ubuntu系统中。然后按照步骤提示安装，其中，选择**最小安装，不连接网络，擦除磁盘安装ubuntu18(不是擦除原来的系统)**。在擦除时会提示删除某个分区，记录下分区名字，我的是**/dev/mmcblk1**(l是L小写）。这里我没有自己分区。安装进度条到3/5左右时，会提示grub2 failed to install to /target/，这时系统其实已经安装成功，可以强制关机。

5. 同样的方法开机，来到U盘的grub引导界面。键盘按e进入到命令行模式。执行ls命令，会显示(hd0),(hd0,gpt1),(hd1,gpt1)等分区设备。执行

   ```shell
   ls (hd1,gpt2)/boot/
   ```

   查看是否有vm\*-\*和init\*这两个文件以及其它大量文件。vm后面是有一长串的版本号的那个。没有的话，换其他设备分区，逐个查找。执行

   ```shell
   set root=(hd1,gpt2)  # 这里hd和gpt后面数字都是对应自己找的设备分区
   linux /boot/vmlinuz-*  root=/dev/mmcblk1p2 # * 表示自己用键盘的tab不全，不是通配符。mmcblk1是刚安装在擦除原来分区时，看见的设备名称。
   initrd /boot/initrd.img 
   boot
   ```

   进入到刚安装的ubuntu18系统。接下来修复grub引导。

6. 连接网络。打开终端，依次执行

   ```shell
   sudo apt-get update
   
   sudo apt-get -y purge grub-efi-amd64 grub-efi-amd64-bin grub-efi-amd64-signed
   
   sudo apt-get -y install grub-efi-ia32-bin grub-efi-ia32 grub-common grub2-common
   
   sudo grub-install --target=i386-efi /dev/mmcblk1p2 --efi-directory=/boot/efi/ --boot-directory=/boot/
   
   # 这里的“mmcblk1p2 ”就是上一步你执行成功的那个值
   
   sudo grub-mkconfig -o /boot/grub/grub.cfg     
   # 然后重启就可以了。
   ```

   倒数第二步可能遇到的错误。

   ​	**/boot/efi似乎不是efi分区**。grub-install: error: /boot/efi doesn't look like an EFI partition.. 执行

   ```shell
   sudo fdisk -l  # 查看efi分区在哪个设备分区。我的是mmcblk1p1
   sudo mount /dev/mmcblk1p1 /boot/efi  # 挂载过去。然后再执行倒数第二步
   ```

   ​      **grub-install：错误： /usr/lib/grub/i386-efi/modinfo.sh 不存在。请指定 --target 或 --directory.**  执行(**没有遇到这个错误，下面的命令是在搜寻时找到的解决参考**)

   ```shell
   cd /tmp
   sudo apt-get download grub-pc-bin
   mkdir grub
   dpkg-deb -R grub-pc-bin_2.02~beta2-36ubuntu3.32_amd64.deb grub/
   sudo mv grub/usr/lib/grub/i386-pc/  /usr/lib/grub/
   # 在grub目录下放入i386-pc文件夹。
   ```

7. 结束，然后重启正常使用。安装ssh服务端，执行

   ```shell
   sudo apt-get install openssh-server # 安装
   sudo /etc/init.d/ssh start # 启动
   ```

   远程客户端需要知道ip，执行

   ```shell
   sudo apt-get install net-tools
   ifconfig # 查看ip
   ```

   后面就开始在远程操作了。

8. 发现触摸屏不能用，还有一个触摸屏驱动安装问题。

# 触摸屏驱动

```shell
git clone GitHub - onitake/gslx680-acpi: ACPI/x86 compatible driver for Silead GSLx680 touchscreens
# https://github.com/onitake/gsl-firmware/tree/master/firmware  一些型号平板的固件。
apt-get install gcc make
cd gslx680-acpi
make
make install
depmod -a

cp silead_ts.fw /lib/firmware
rmmod silead
rmmod gslx680_ts_acpi
modprobe gslx680_ts_acpi
# 然后重启试下
```



# 参考网页

1.https://www.bilibili.com/read/cv18254808     X86平板安装Ubuntu

2.https://xuexiyuan.cn/article/detail/149.html 引导错误，需要挂载

3.https://blog.csdn.net/danshiming/article/details/122765324 /usr/lib/grub/i386-pc/modinfo.sh doesn‘t exist错误

4.https://greyishsong.ink/平板的折腾之旅：从Windows 10到Ubuntu/   这里提到触摸屏的驱动问题

5.https://www.zhihu.com/question/38587635 触摸屏驱动

ps:里面可能还有细节忘了。太难折腾了。








# Manjaro 安装配置

title: Manjaro 安装配置
date: 2020-01-21 19:32:30
tags: [Linux, Manjaro] 
categories: [技术]

------




非常感谢这篇[博客](https://printempw.github.io/setting-up-manjaro-linux/)提供的思路，让我在后面的安装过程中少走了不少弯路。




整理下我们目前的思路

- 下载镜像文件
- 制作启动盘
- 安装linux系统
- 调试和使用

目标安装机器配置：

| -    | -                    |
| ---- | -------------------- |
| 型号 | Lenovo Erazer Y40-70 |
| CPU  | Intel i7-4510U 2G    |
| HDD  | 1T                   |
| 内存 | 4GB                  |
| LAN  | 100M/1000M           |
| OS   | Windows 8.1 PRC      |




## 下载镜像文件

这个简单，到[官网](https://manjaro.org)下就行了

我下载的是[gnome版本](https://manjaro.org/download/official/gnome/)的

全名为` manjaro-gnome-18.1.5-191229-linux54.iso `

SHA1为` 7182126e8d2a103024028c3b242ac49535e2ea00 `

下载了将近两个小时

```powershell
> Get-FileHash .\manjaro-gnome-18.1.5-191229-linux54.iso
...
SHA1            7182126E8D2A103024028C3B242AC49535E2EA00                               
```



## 制作启动盘

这里使用的工具是[Rufus](https://rufus.ie/)

[注意事项](https://manjaro.org/support/firststeps/#making-a-live-system)

- 选择dd模式
- 用来做启动盘的U盘中原来的数据会丢失

但是我制作的时候没有提示要求选择dd模式，比较费解

一般来说需要备份下原来的系统，这里我不关心原来的系统（爆炸去吧）直接上路

### 从usb启动

准备工作，需要先从系统设置中关闭 Secure Boot       

否则会报"efi usb device has been blocked by the current security policy"错误

对于win8来说，具体如下

-  进入 控制面板->硬件和声音->电源选项->系统设置
- 点击 更改当前不可用的设置
- 在关机设置中，取消"启动快速启动"

重启电脑，按F2进入BIOS，选择`Security`选项卡

光标移动到`Secure Boot`选项后，回车改变对应值

之后F10退出并报错

再次启动后按F12选择 从 EFI USE Device 启动

如果没问题的话，你能看到`Welcome to Manjaro`的提示，之后会自动重启安装。

之后有了图形界面，弹出欢迎页面，就可以用图形化的方式来安装了。

具体流程可以参考[此处](https://www.jianshu.com/p/7cfc16dba667)。



### 联网

安装的步骤有些组件需要联网，本来以为连个WI-FI还不容易，结果花了许多时间来穷举配置。

当前的网络协议EAP,此时连接配置选择如下

- Wi-Fi安全性 WPA及WPA2企业
- 认证 受保护的EAP(PEAP)
- 不需要CA证书
- 内部认证 MSCHAPv2



## 调试和使用

```shell
# 设置国内镜像源
$ sudo pacman-mirrors -c China -i -m rank
# 更新自带软件包
$ sudo pacman -Syu
```

听说Aur是Arch Linux很有特色的一个东西，下面的命令先抄过来配下，看看好不好用

```shell
# 后面那个是编译包时需要的一些工具，不然会报错缺少 fakeroot 之类的
sudo pacman -S yay base-devel
# 设置 AUR 清华镜像源
yay --aururl "https://aur.tuna.tsinghua.edu.cn" --save
```



pacman部分指令速查表

```shell
pacman -Q 																	  # 查看所有安装的包
pacman -S package_name1 package_name2 ...     # 安装软件
pacman -R package_name                        # 删除软件
pacman -Syu                                   # 更新软件和系统
pacman -Ss string1 string2 ...                # 搜索
```



如果你觉得同时操作多个电脑很麻烦，并且有跨平台的需求（比如，一台是windows，一台是linux）

那么可以使用这个软件[Barrier](https://github.com/debauchee/barrier)



图文教程可以参考[此处](https://ywnz.com/linuxjc/5776.html )

配置完成后，我就能用一套键盘/鼠标来控制两台不同电脑上的不同操作系统了。

操作手感和双屏差不多，还能支持剪贴板的复制粘贴。

我这里就没配置开机启动了，因为DHCP分配到的IP地址并不固定，每次得确认下。

可惜不能支持文件的互传。



其他调试

```shell
# 为了能使用ifconfig 安装了这个包
$ sudo pacman -S net-tools

# 开启ssh服务， 这样我在别的电脑能直接ssh过去
$ systemctl enable sshd.service
$ systemctl start sshd.service
$ systemctl status sshd.service
```

后续可以考虑做个内网穿透，放在家里做服务器。

### 安装中文输入法

安装后默认是没有中文输入法，所以这里需要我们自己安装。

```shell
$ sudo pacman -Syu fcitx fcitx-googlepinyin fcitx-im fcitx-configtool
$ sudo pacman -Syu vim

# 以下部分参照
#   https://wiki.archlinux.org/index.php/Fcitx_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)
# 设置环境变量
# 将下面内容加入桌面的启动脚本，以注册输入法模块并支持 xim 程序。

$ vim ~/.pam_environment
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
```

这里我直接配置完了，但是不能切换输入法，先用

` fcitx-diagnose `诊断问题所在

```shell
$ fcitx-diagnose > log
$ vim log
```

按照诊断的建议，我把上面在`~/.pam_environment`的配置复制到了` ~/.xprofile`

重启后发现还是不行，怀疑是无法读取环境变量，做如下处理,在这里添加环境变量

```shell
$ sudo vim /etc/environment
```

这下终于好了

<del>一个配置文件我写了三个地方 </del>大草




---
layout: post
title:  "安信可M62-CBS模组（BL616 芯片）用作 SDIO / USB 无线网卡和 BLE 适配器"
date:   2024-04-07 17:00:14 +0800
categories: iot bouffalolab bl616
---

# 一、概述

安信可 M62-CBS 无线网卡模组采用博流 BL616 芯片方案，提供 SDIO / USB 接口，同时可以通过 UART / USB 接口作为 BLE HCI 适配器使用。本文以树莓派为例，介绍 M62-CBS 驱动移植及联网的过程。

# 二、软硬件要求
## 1. 目标板硬件要求
### 1.1. 通过 USB 接口连接

- USB接口

### 1.2. 通过 SDIO 接口连接

- SDIO
- UART（若需要使用 BLE 功能）

## 2. 目标板软件要求

- Linux Kernel 3.10 ~ 5.15.79（需要启用 mac80211 相关配置选项）
- wpa_supplicant / hostapd 2.9+ （需要启用 nl80211 选项后编译）
- BlueZ 5 （若需要使用 BLE 功能）

## 3. 开发机软件要求

- 目标板 Linux 内核源码
- 目标板交叉编译工具链

# 三、硬件连接

您的 Linux 开发板可以通过 USB 接口或者 SDIO + UART 接口连接到 M62-CBS 模组。为了方便用户调试使用，我们已经预先设计好了下图所示的转接板，插上即可完成连接。对应嘉立创工程文件放在文章末尾的附件内了，打样时注意板厚设置为 0.8mm，不然插不进 SD 卡槽中。

![M62-CBS TF 卡转接板]({{ '/assets/images/m62-cbs-sdio-adapter.jpeg' | relative_url }})

# 四、驱动编译

驱动 Makefile 默认使用 Native 工具链编译：

![Makefile 修改 1]({{ '/assets/images/bl616-driver-modification-1.png' | relative_url }})

交叉编译需要修改`KERNELDIR` 变量，取消 `ARCH` 和 `CROSS_COMPILE` 的注释并修改为实际值。另外还需要交换注释 make 那一行：

![Makefile 修改 2]({{ '/assets/images/bl616-driver-modification-2.png' | relative_url }})

编译出现报错，请仔细对照错误信息和文档检查是否启用了内核相关配置选项，例如 `bl_get_wireless_stats()` 函数报错是因为没有启用 `CONFIG_WIRELESS_EXT` 选项。

具体步骤请参考安信可官网文档《BL616 Linux 驱动移植及使用说明》以及驱动源码下的 README 文档。

# 五、安装驱动到目标板

驱动支持 STA / AP、STA + AP 和 Repeater 三种模式，我们以 STA + AP 共存模式为例子讲解过程，其他两种模式的配置参数可参考源码下的 README 文档。

1. 将内核模块（`bl_sdio_drv.ko` 或 `bl_usb_drv.ko`）和对应模组固件拷贝到目标板文件系统中，将对应模组固件（`bl616_sd_combo.bin` 或 `bl616_usb_combo.bin`）放在 `/lib/firmware` 文件夹下。
2. 执行`insmod bl616_sd_combo.bin opmode=2` 或者 `insmod bl616_usb_combo.bin opmode=2` 加载内核模组。
3. 使用 `ip addr` 命令查看是否新增了 2 块无线网卡，`wlan0` 和 `ap0`，若没有检测到无线网卡，检查接线或者使用 `dmesg` 命令查看内核日志诊断原因
4. 若启用了 `rfkill` ，需要使用 `rfkill unblock all` 启用无线网卡。

# 六、使用 wpa_supplicant 连接 Wi-Fi 热点

1. 使用 `ps -eF | grep wpa_supplicant` 命令检查当前是否运行了其他的 `wpa_supplicant`，若存在需要先 kill 掉，否则会失败。
2. 使用下面指令设置 Wi-Fi 网络

```shell
wpa_passphrase ABC 12345678 > bf.conf  # 把 ABC 和 12345678 替换为你实际WiFi的ssid和password，
wpa_supplicant -D nl80211 -i wlan0 -c bf.conf -B # 启动 wpa_supplicant 连接网络，wlan0换成实际无线网卡接口
udhcpc -i wlan0  # 通过 DHCP 获取 IP 地址
```
3. 连接完成后，使用 `ip addr` 检查是否获取到 IP 地址

4. 连接 WPA3 加密的 WiFi 热点

```shell
cat <<'EOF' > bf.conf
ctrl_interface=/var/run/wpa_supplicant
update_config=1

network={
    ssid="ASUS_2.4"
    scan_ssid=1
    key_mgmt=SAE
    psk="88888888"
    ieee80211w=1
}
EOF
```

# 七、使用 hostapd 开启 Wi-Fi 热点

1. 查看是否存在名称为 `ap0` 的网卡
```shell
ip addr
```

2. 新建 hostapd 配置文件

```shell
cat <<'EOF' > hostapd.conf
interface=ap0
drvier=nl80211

# Operation mode a = IEEE 802.11a (5 GHz), b = IEEE 802.11b (2.4 GHz),
# g = IEEE 802.11g (2.4 GHz), ad = IEEE 802.11ad (60 GHz); 
# a/g options are used with IEEE 802.11n (HT), too, to specify band. 
# For IEEE 802.11ac (VHT), this needs to be set to hw_mode=a. 
# For IEEE 802.11ax (HE) on 6 GHz this needs to be set to hw_mode=a.
hw_mode=g

# 2.4GHz的频段下信道可选1-13
channel=7

wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP

#你的wifi名字
ssid=BL616

#你的密码
wpa_passphrase=12345678
EOF
```

3. 运行 hostapd 服务

```shell
hostapd -B hostapd.conf
```

4. 配置 AP 的 IP 地址

```shell
ip address add 192.168.9.1/24 dev ap0
```

5. 配置 AP 的 DHCP 服务

```shell
cat <<'EOF' > udhcpd.conf
interface ap0
start 192.168.9.100
end 192.168.9.200
max_leases 200
EOF
```
6. 启动 DHCP 服务器

```shell
udhcpd udhcpd.conf & 
```

7. 连接热点进行测试，SSID `BL616`，密码 `12345678`

# 八、BLE

1. 初始化蓝牙（仅 UART 方式，USB可以跳过）

```shell
# /dev/ttyUSB0：模组接到树莓派的串口号，根据实际串口号替换
hciattach -s 115200 /dev/ttyUSB0 any 115200 noflow nosleep
```

2. 查询 HCI

```shell
hciconfig
```

若出现 `hci0` 等设备代表 OK。

3. 开启和重启一下模组蓝牙

```shell
hciconfig hci0 up
hciconfig hci0 reset
```

4. （可选）配置 BLE 广播数据，可以在调试助手上点击扫到的 BLE 设备查看

```shell
hcitool -i hci0 cmd 0x08 0x0008 1f 02 01 06 0b 09 42 54 42 4c 45 2d 44 45 56 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

5. （可选）配置 BLE 扫描响应数据，可以在调试助手上点击扫到的 BLE 设备查看

```shell
hcitool -i hci1 cmd 0x08 0x0009 1f 02 01 06 0a 09 42 54 42 4c 45 2d 44 45 56 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

6. 开启 BLE 广播

```shell
hciconfig hci1 leadv 0
```

7. 打开监控软件

```shell
btmon
```

8. 停止广播和再次广播

```shell
hciconfig hci1 noleadv
hciconfig hci1 leadv 0
```

# 九、更多资料

- [Ai-M62系列模组专题](https://docs.ai-thinker.com/ai_m62)
- [嘉立创安信可科技开源团队](https://oshwhub.com/ai-thiner_openteam)


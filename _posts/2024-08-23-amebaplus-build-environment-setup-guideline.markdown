---
layout: post
title:  "安信可 BW20-12F / BW20-07S 模组（RTL8711DAx）开发环境搭建指南"
date:   2024-08-23 15:51:36 +0800
categories: iot realtek amebadplus
---

# 一、概述

BW20-12F 和 BW20-07S 是安信可基于 RTL8711DAx 芯片开发的双频 Wi-Fi 4 + BLE 5.0 SoC 模组，支持双频 802.11a/b/g/n WLAN 协议和 BLE 5.0 协议。BW20-12F 集成了双核 MCU，一个兼容 Cortex-M55 的高性能 MCU，最高主频可达 330MHz；一个兼容Cortex-M23 的低功耗 MCU。

BW20 系列模组具有丰富的外设接口，包括 UART / GPIO / ADC / PWM / I2S / I2C / SPI / SDIO / IR / SWD / USB 等。可广泛应用于物联网（IoT）、移动设备、可穿戴电子设备、智能家居等领域。

本文讲述如何基于 BW20 系列模组进行软件开发，包含环境搭建、编译、烧录和调试等内容。

![BW20-12F 正面图片]({{ '/assets/images/bw20-12f-top-view.png' | relative_url }})
![BW20-07S 正面图片]({{ '/assets/images/bw20-07s-top-view.png' | relative_url }})

# 二、编译环境搭建

瑞昱 RTL8711DAN 的 SDK 基于 FreeRTOS，开发和编译需要在 Linux 下进行，烧录软件需要在 Windows 下运行。

下面以 Ubuntu 24.04 LTS 为例，讲解如何搭建瑞昱 RTL8711DAN 的开发环境。

## 1. 安装依赖

```shell
sudo apt install -y build-essential libncurses-dev libc6-i386 git-core virtualenv
```

## 2. 配置 Git

```shell
git config --global user.name 你的用户名
git config --global user.email 你的电子邮件地址
```

## 3. 配置 Git 使用 HTTP 代理服务器

```shell
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
```

瑞昱 RTL8711DAN 的 SDK 在 GitHub 上，国内访问 GitHub 受阻，因此需要配置 HTTP 代理服务器。将上面命令的 `127.0.0.1` 替换成实际代理服务器地址，`7890` 替换为实际代理服务器端口号。

## 4. 配置 pip 使用 HTTP 代理服务器

在 `$HOME/.profile` 末尾新增下面的命令，或者每次打开 shell 时执行：

```shell
export HTTP_PROXY="http://127.0.0.1:7890"
export HTTPS_PROXY=$HTTP_PROXY
export http_proxy=$HTTP_PROXY
export https_proxy=$HTTPS_PROXY
```

同样的，将上面命令的 `127.0.0.1` 替换成实际代理服务器地址，`7890` 替换为实际代理服务器端口号。

## 5. 克隆 SDK 到本地

运行下面的命令，使用 `git` 命令克隆 `ameba-rtos` 仓库到本地：

```shell
cd ~
git clone https://github.com/Ameba-AIoT/ameba-rtos.git -b release/v1.0
```

## 6. 配置 virtualenv 隔离 Python 运行环境

SDK 的编译脚本依赖一些 Python 包，为了避免干扰到其他 Python 程序的运行，需要创建一个隔离的 Python 运行环境。

运行下面的命令初始化一个 virtualenv 环境，名称 `venv`：

```shell
cd ~
virtualenv venv
source ~/venv/bin/activate
```

运行完成后，可以看见 shell 变成下图所示：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b8d75db7254c432abe041c48f066d260.png)

在 virtualenv 环境中使用 `pip` 安装 SDK 需要的 Python 依赖：

```shell
cd ~/ameba-rtos
pip3 install -r tools/image_scripts/requirements.txt
```

## 7. 编译

每次编译前都需要加载 virtualenv 环境：

```shell
source ~/venv/bin/activate
```

运行下面命令进入 menuconfig 配置界面：

```shell
cd ~/ameba-rtos/amebadplus_gcc_project
make CONFIG_SHELL=/bin/bash menuconfig
```

运行下面命令编译固件：

```shell
make CONFIG_SHELL=/bin/bash
```

编译完成后会在 `~/ameba-rtos/amebadplus_gcc_project/` 目录下生成 `km4_boot_all.bin` 和 `km0_km4_app.bin` ，这两个就是最后烧录到模组中的固件。



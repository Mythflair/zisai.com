---
title: "BeagleBone Black 开发笔记：串口 TCP 数据采集与 GPIO 告警控制"
date: 2025-06-15T14:05:00+09:00
draft: false
tags: ["BeagleBone Black", "嵌入式开发", "串口", "GPIO", "Debian"]
categories: ["技术分享", "嵌入式系统"]
description: "记录使用 BeagleBone Black 实现串口服务器 TCP 数据采集、屏幕输出及 GPIO 控制红黄绿灯的开发过程，分享踩坑经验与解决方案。"
cover:
    image: "/images/v2-1c7cfd932d637812f57f33699360243e_1440w.png" 
    alt: "BeagleBone Black 开发笔记：串口 TCP 数据采集与 GPIO 告警控制"
---

# BeagleBone Black 开发笔记：串口 TCP 数据采集与 GPIO 告警控制

BeagleBone Black（以下简称 BBB）是一款功能强大的单板计算机，尽管用户群体较树莓派小，但其响应速度和灵活性在特定场景下颇具优势。本文记录了我近期使用 BBB 开发一个项目的经历：通过 TCP 协议采集串口服务器数据，输出到屏幕并标记告警信息，同时通过 GPIO 控制红黄绿三色告警灯。以下分享开发过程中的踩坑经验和解决方案，希望对使用 BBB 的开发者有所帮助。

## 项目背景与硬件选择

项目目标是从串口服务器接收 TCP 数据，实时显示并标记告警，同时通过 GPIO 输出控制三色灯。BBB 搭载 AM3358 处理器，内置 4GB eMMC 和 512MB 内存。虽然树莓派的 2GB 内存和更强 GPU 更为主流，但 BBB 在实时响应上表现更优。考虑到我对树莓派和 CentOS 较熟悉，对 Debian 了解有限，BBB 的开发过程充满挑战。

以下是开发中的关键关注点与解决方案。

## 关注点一：按键误标与 MicroSD 引导误区

初次接触 BBB 时，我误以为其未预装系统，尝试按树莓派方式从 MicroSD 卡引导。下载了 TDA4VM Debian 11.3（2022-06-14）镜像，参照网上资料按住 S2 键（Boot 键）启动，却发现四个状态灯未同时亮起，表示未从 SD 卡引导。反复尝试无果，甚至怀疑板载系统有问题。

**问题分析**：
- **按键误标**：网上部分资料将 S2（Boot）键误标为 S3，导致混淆。实际板上印刷清晰，S1=Reset，S2=Boot，S3=Power。
- **镜像不匹配**：TDA4VM 镜像是为 BeagleBone AI-64 设计的，与 BBB 的 AM3358 硬件不兼容，引导失败。

**解决方案**：下载正确的 AM3358 Debian 10.3（2020-04-06）IoT 镜像，确认硬件匹配后，BBB 板载系统（Debian 10.3）可正常启动，MicroSD 卡并非必需。

## 关注点二：镜像烧录与版本选择

烧录镜像时，我最初直接使用 `.xz` 文件通过 Win32DiskImager 写入 SD 卡，导致引导失败。

**问题分析**：
- `.xz` 文件需解压为 `.img` 文件再烧录。
- BBB 镜像分为“Recommended”和“Flasher”版本：
  - **Recommended**：可通过 SD 卡引导运行，修改 `/boot/uEnv.txt`（取消 `cmdline=init=/opt/scripts/tools/eMMC/init-eMMC-flasher-v3.sh` 的注释）后可烧录到 eMMC。
  - **Flasher**：自动将镜像烧录到 eMMC，适合快速部署。

**解决方案**：
1. 下载 AM3358 Debian 10.3（2020-04-06）4GB SD IoT 镜像，解压 `.xz` 为 `.img`。
2. 使用 Win32DiskImager 烧录至 MicroSD 卡。
3. 插入 SD 卡，按住 S2 键启动，烧录至 eMMC：
   ```bash
   cd /opt/scripts/
   git pull
   sudo ./update_kernel.sh
   sudo reboot
   ```
4. 使用阿里云镜像源加速更新：
   ```bash
   sudo vim /etc/apt/sources.list
   ```
   添加以下内容到文件头部：
   ```
   deb https://mirrors.aliyun.com/debian/ buster main non-free contrib
   deb-src https://mirrors.aliyun.com/debian/ buster main non-free contrib
   deb https://mirrors.aliyun.com/debian-security buster/updates main
   deb-src https://mirrors.aliyun.com/debian-security buster/updates main
   deb https://mirrors.aliyun.com/debian/ buster-updates main non-free contrib
   deb-src https://mirrors.aliyun.com/debian/ buster-updates main non-free contrib
   deb https://mirrors.aliyun.com/debian/ buster-backports main non-free contrib
   deb-src https://mirrors.aliyun.com/debian/ buster-backports main non-free contrib
   ```
   然后执行：
   ```bash
   sudo apt-get update
   sudo apt-get upgrade
   ```

**注意**：eMMC 容量仅 4GB，建议使用 32GB 以上 MicroSD 卡运行生产环境，以避免空间不足。

## 关注点三：系统登录与网络配置

BBB 默认无 WiFi 模块，需通过 USB 或网线连接。

**登录方式**：
- **USB 连接**：连接电脑后约 10 秒，系统识别为移动设备，可通过 SSH 访问 `192.168.7.2`，使用 `debian/temppwd` 登录（root 账户默认禁用）。也可通过浏览器访问 `http://192.168.7.2`。
- **网线连接**：插入网线后，系统通过 DHCP 分配 IP（如 `192.168.3.180`），可通过路由器查询。

**网络问题**：USB 供电联网时，更新常失败。插入网线后，网络稳定，更新顺畅。

## 关注点四：系统初始化与软件更新

进入系统后，需进行初始化配置以确保稳定运行。

1. **设置时区**：
   ```bash
   sudo timedatectl set-timezone 'Asia/Shanghai'
   ```

2. **更新脚本与内核**：
   ```bash
   cd /opt/scripts/
   git pull
   cd tools/
   sudo ./update_kernel.sh
   sudo reboot
   ```

3. **更新软件包**：
   ```bash
   sudo apt-get update
   sudo apt-get upgrade
   ```
   升级过程中，`c9-core-installer` 包可能无法更新，需卸载后重装：
   ```bash
   sudo apt-get purge c9-core-installer
   sudo apt-get install c9-core-installer
   ```

4. **LSB 模块问题**：
   执行 `lsb_release -a` 显示 `No LSB modules are available`，虽无大碍，但可通过 `lsb_release -cdir` 避免提示。不建议安装过时的 `lsb-core` 包。

## 关注点五：MicroHDMI 输出问题

板载系统无法通过 MicroHDMI 输出视频信号，尝试调整分辨率、安装驱动、修改 EDID 配置均无效。

**解决方案**：刷写官方 AM3358 Debian 10.3 镜像后，MicroHDMI 输出正常，甚至在树莓派 10.1 寸屏上无需额外驱动即可显示。不同显示器可能因分辨率不匹配导致显示不全，建议测试前确认兼容性。

## 项目进展与建议

目前，BBB 已成功配置为接收串口服务器的 TCP 数据并通过 GPIO 控制三色灯。后续需编写 Python 或 C 程序处理 TCP 数据流、解析告警信息并驱动 GPIO。具体实现代码将在后续开发中补充。

**建议**：
- **优先使用官方镜像**：确保硬件与镜像版本匹配，避免引导失败。
- **选择合适存储**：eMMC 容量有限，生产环境推荐 MicroSD 卡。
- **优化网络环境**：使用网线连接以确保更新稳定，阿里云镜像源可显著提升速度。
- **调试 HDMI 输出**：如遇显示问题，直接刷官方镜像，避免无效调试。

## 结语

BeagleBone Black 的开发充满挑战，从按键误标到镜像烧录，每一步都可能踩坑。但通过耐心试错和查阅资料，这些问题都能迎刃而解。这次经历让我重新拾起嵌入式开发的乐趣，也提醒我：技术探索需要细心与坚持。

*本文改编自2023年3月25日的个人技术记录。*
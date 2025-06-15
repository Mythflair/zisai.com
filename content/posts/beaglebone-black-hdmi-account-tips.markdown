---
title: "BeagleBone Black 补充笔记：MicroHDMI 与账号配置"
date: 2025-06-15T14:15:00+09:00
draft: false
tags: ["BeagleBone Black", "嵌入式开发", "MicroHDMI", "账号配置", "Debian"]
categories: ["技术分享", "嵌入式系统"]
description: "补充 BeagleBone Black 的 MicroHDMI 接口调试和账号配置经验，解决视频输出与 root 远程登录问题。"
cover:
    image: "/images/pr_01_1900.jpg" 
    alt: "BeagleBone Black 补充笔记：MicroHDMI 与账号配置"
---

# BeagleBone Black 补充笔记：MicroHDMI 与账号配置

在 BeagleBone Black（以下简称 BBB）的开发过程中，MicroHDMI 接口和账号配置是两个常见但易被忽略的细节。继之前的系统配置和 Go 环境搭建后，本文分享了解决 MicroHDMI 无输出问题和启用 root 远程登录的经验，以及 MicroSD 卡空间扩展的简单方法。这些碎片化记录为 BBB 开发者提供实用参考。

## 解决 MicroHDMI 输出问题

BBB 的 MicroHDMI 接口常出现无视频输出问题，尤其在使用转接器时。以下是调试经验：

**问题分析**：
- **直接连接推荐**：优先使用 MicroHDMI 转 HDMI 或 MicroHDMI 转 DVI 线，匹配显示器或显卡接口，确保直接点亮屏幕。
- **转接器电流问题**：若转接到 VGA 等老式接口，需注意 BBB 的保险丝（RT1，型号 RXEF010，黄色滴胶状，双脚）。其保持电流为 100mA，短路电流为 200mA。VGA 转接器输出电流常超过 200mA，导致保险丝进入高阻态，切断 HDMI 供电，致无视频信号。
- **保险丝恢复**：RT1 为自恢复保险丝，理论上断开负载后可恢复，但实际恢复情况需视硬件状态而定，建议测试前检查。

**解决方案**：
1. 使用 MicroHDMI 转 HDMI/DVI 线，避免 VGA 转接。
2. 若必须使用 VGA，确认转接器电流低于 200mA。
3. 如仍无输出，刷写官方 AM3358 Debian 10.3 镜像（参考前文），确保系统兼容性。

## 配置账号与远程登录

BBB 默认提供两个账号：
- `debian`：密码为 `temppwd`，支持 SSH 登录。
- `root`：密码为 `root`，默认禁用远程 SSH 登录，且 `sudo` 需输入密码。

为方便开发，我启用 `root` 账号的远程 SSH 登录。

**步骤**：
1. 编辑 SSH 配置文件：
   ```bash
   vim /etc/ssh/sshd_config
   ```
   添加：
   ```
   PermitRootLogin yes
   ```
2. 重启 SSH 服务：
   ```bash
   sudo systemctl restart sshd
   ```
3. 修改 `root` 密码（推荐）：
   ```bash
   passwd root
   ```

## 扩展 MicroSD 卡空间

BBB 的板载 eMMC 仅 4GB，空间有限。使用 MicroSD 卡运行系统时，可扩展分区以充分利用存储。

**步骤**：
1. 运行扩展脚本：
   ```bash
   cd /opt/scripts/tools
   sudo ./grow_partition.sh
   sudo reboot
   ```
2. 验证分区扩展：
   ```bash
   fdisk -l
   df -h
   ```

## 注意事项

- **MicroHDMI 调试**：避免盲目修改 EDID 或驱动配置，优先尝试官方镜像刷机。
- **账号安全**：启用 `root` 远程登录后，务必设置强密码，避免安全风险。
- **存储管理**：生产环境建议使用 32GB 以上 MicroSD 卡，以支持复杂应用和环境配置。

## 结语

MicroHDMI 输出、账号配置和存储扩展是 BeagleBone Black 开发中的常见问题。通过选择合适的接口、启用 root 登录和扩展 MicroSD 空间，这些问题都能轻松解决。这些经验为后续 TCP 数据采集和 GPIO 控制开发奠定了基础。BBB 的开发虽有挑战，但乐趣无穷，期待更多探索！

*本文改编自2023年3月25日的个人技术记录。*
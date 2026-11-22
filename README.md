# 360T7 固件构建（ImmortalWrt mt798x 6.6 + WireGuard）

为 **360 T7（MT7981）** 构建带 WireGuard 的固件，源码来自
[`padavanonly/immortalwrt-mt798x-6.6`](https://github.com/padavanonly/immortalwrt-mt798x-6.6)
分支 `mt798x-mt799x-6.6-mtwifi`（6.6 内核 + MTK `mt_wifi` 闭源驱动）。

## 为什么用这个分支

原来那套 `24.10-SNAPSHOT r33079-53c48bbd7e`（target `mediatek/mt7981`、内核 5.4.284）
的上游已经废弃：`mt7981` 子目标被 ImmortalWrt 官方从源码树里移除，个人 fork
`Xiaoze886/immortalwrt-mt798x-24.10` 自 2025-07-20 起停更，`kmod-wireguard` 无处可装。

本分支把同一批设备迁到 6.6 内核，WireGuard 回到主线，`DEFAULT_PACKAGES` 里已自带
`kmod-wireguard`。

## 构建内容

- 目标：`mediatek/filogic`，设备 `qihoo_360t7`
- 额外预装：`wireguard-tools`、`luci-proto-wireguard`、`luci-app-wireguard`、中文语言包
- 产物：`*sysupgrade.itb`、`*recovery.itb`、`bl31-uboot.fip`、`preloader.bin`、相关 ipk

## 触发方式

仓库 → Actions → `Build 360T7 Firmware` → Run workflow。首次约 2–3 小时。

构建完成后在 run 页面底部下载 `360t7-firmware` artifact。

## 预装清单

| 类别 | 内容 |
|---|---|
| WireGuard | `kmod-wireguard`、`wireguard-tools`、`luci-proto-wireguard`、`luci-app-wireguard` |
| 上网管理 | `appfilter`（应用/网站过滤）、`banip`（黑名单）、`adblock`、`https-dns-proxy` |
| 时间与限速 | `wifischedule`（定时关 Wi-Fi）、`nft-qos`（按设备限速）、`sqm`（抗缓冲） |
| 设备管控 | `arpbind`（IP-MAC 绑定）、`nlbwmon`（每设备流量统计） |
| 代理 | `luci-app-openclash` |
| 运维 | `ttyd`（网页终端）、`attendedsysupgrade`（一键升级）、`package-manager`、`ddns` + cloudflare、`vlmcsd` |
| 工具 | bash、curl、wget-ssl、htop、nano、socat、tcpdump、ip-full、ipset、zram-swap |

## 和旧固件相比会失去什么

旧固件是 MTK SDK 那套（`mediatek/mt7981` + 5.4 内核），下列包**依赖 MTK 闭源驱动 `mt_wifi`，在 6.6/filogic 上不存在**：

- `luci-app-mtwifi-cfg`（MTK 无线高级参数面板）
- `luci-app-turboacc-mtk`（MTK 硬件加速开关）→ 用「防火墙 → 软件/硬件流量分载」替代
- `luci-app-eqos-mtk`（MTK QoS）→ 用 `luci-app-nft-qos` 替代
- `luci-app-upnp-mtk-adjust`、`mtkhqos_util`、`mtk-smp`、`mhz`、`regs`

360T7 在新分支里走 **mt76 开源驱动**（`kmod-mt7915e`），不是 `mt_wifi`。

## 刷机前必读

1. **先备份全盘**：`dd if=/dev/mtd0 of=/tmp/mtd0.bin`，把 128MB 完整备份拉到电脑上。
2. 先刷 `*recovery.itb`（initramfs，跑在内存里，不写 flash）验证 Wi-Fi 和网口正常。
3. 确认无误再写 `sysupgrade.itb`。
4. 该设备 profile 使用 `KERNEL_IN_UBI` / `UBOOTENV_IN_UBI`，bootloader 与当前
   原厂 uboot 可能不兼容；若 recovery 起不来，走 TFTP 恢复，不要硬刷。

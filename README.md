# 360T7 固件构建（ImmortalWrt 25.12 · 刷不死 U-Boot 布局）

为 **360 T7（MT7981，128MB SPI-NAND，256MB 内存）** 构建带 WireGuard 和上网管理的固件。

- 源码：官方 [`immortalwrt/immortalwrt`](https://github.com/immortalwrt/immortalwrt) tag `v25.12.2`（内核 6.12，apk 包管理）
- 设备：`mediatek/filogic` → `qihoo_360t7-ubootmod`（custom U-Boot layout）
- 刷机方式：刷不死 U-Boot 的恢复网页，**不碰 BL2 / FIP**

## 适用前提

这个固件只适用于 FIP 分区里装的是**魔改"刷不死"U-Boot**（hanwckf bl-mt798x 一类，带网页恢复）的机器。
在当前系统里确认：

```sh
strings /dev/mtd4 | grep -iE "U-Boot 20|failsafe"   # 应看到 "Web failsafe UI started"
cat /proc/mtd                                       # ubi 应为 06c00000（108M），起始 0x580000
```

原厂 U-Boot 的机器**不要**用这个固件。

## 为什么是 ubootmod，而不是官方的 `qihoo_360t7`

官方的 `qihoo_360t7` / `qihoo_360t7-ubi` 用的是 OpenWrt 自己的 U-Boot（内核以 FIT 形式放在 UBI 卷里，环境变量也存在 UBI 里），
装它就**必须改写 BL2 和 FIP**：
- 会覆盖掉刷不死 U-Boot，网页恢复功能就没了；
- 写坏了只能拆机接 TTL 串口救。

`ubootmod` 布局沿用刷不死 U-Boot 认得的经典 UBI 三卷（kernel / rootfs / rootfs_data），分区表和原机一致，
从恢复网页直接刷，刷坏了可以随时重刷。

ImmortalWrt 23.05 曾官方提供 `qihoo_360t7-ubootmod`，24.10 起去掉了。这里的设备树和设备定义（[`ubootmod/`](ubootmod/)）取自
[dontshootitsme/immortalwrt-25.12.x-…-qihoo_360t7-ubootmod-squashfs](https://github.com/dontshootitsme/immortalwrt-25.12.x-mediatek-filogic-qihoo_360t7-ubootmod-squashfs)
（commit `1c204a7`），由 23.05 官方定义移植到 25.12，与官方 23.05 一样开启 NMBM。CI 额外修补了 `02_network`，
让网口配置为 `lan1 lan2 lan3` + `wan`（否则会多出一个不存在的 `lan4`）。

## 预装清单

| 类别 | 内容 |
|---|---|
| WireGuard | `kmod-wireguard`、`wireguard-tools`、`luci-proto-wireguard` |
| 上网管理 | `appfilter`（应用/网站过滤）、`banip`（黑名单）、`adblock`、`https-dns-proxy` |
| 时间与限速 | `wifischedule`（定时关 Wi-Fi）、`eqos`（按设备限速）、`sqm`（抗缓冲，cake） |
| 设备管控 | `arpbind`（IP-MAC 绑定）、`nlbwmon`（每设备流量统计） |
| 代理 | 只预装 OpenClash 需要的内核模块：`kmod-tun`、`kmod-nft-tproxy`、`kmod-inet-diag`（另有 `dnsmasq-full`）。OpenClash 本体刷机后装，见下文 |
| 运维 | `ttyd`（网页终端）、`package-manager`、`ddns` + cloudflare、`vlmcsd` |
| 工具 | bash、curl、wget-ssl、htop、nano、socat、tcpdump、ip-full、ipset、zram-swap、block-mount、dnsmasq-full |

和 24.10 时期清单相比的变化：
- `luci-app-nft-qos` 在 25.12 源里已没有，改用 `luci-app-eqos`；
- 去掉了 `attendedsysupgrade`：在线升级服务器不认识自编的 ubootmod profile，用不了；
- 25.12 用 **apk** 代替 opkg：`apk update`、`apk add <包>`。
- OpenClash 不再编进固件（它依赖 ruby，ruby 默认的 YJIT 要编 Rust，而 Rust 构建要下载的 LLVM 已被上游删除）。刷机后安装：
  ```sh
  apk update
  apk add luci-app-openclash
  ```
  ruby 等用户态依赖从官方源拉；它需要的内核模块已预装，不受 vermagic 影响。也可以用
  [OpenClash Releases](https://github.com/vernesong/OpenClash/releases) 里的 `.apk`。

## 编译缓存

CI 用 `actions/cache` 保存 `dl/`（源码包）和 ccache（编译缓存），**失败或超时也会保存**，下次从最近一份恢复。
首次运行没有缓存；之后只改包清单的话，大部分文件会命中缓存。命中率见 “ccache stats” 一步。
编译步骤单独限时 315 分钟，比整个任务的 355 分钟早结束，留时间保存缓存。

## 内核模块与官方源一致（vermagic）

默认（`match_official_kmods = true`）用官方 buildbot 的 `config.buildinfo` 作种子配置（保留内核选项和 `ALL_KMODS`），
目标是让内核 vermagic 与官方 25.12.2 一致，这样刷机后可以直接 `apk add` 官方源里的任意 `kmod-*`。
构建结束时 CI 会比对，结果写在 “Check images and vermagic” 一步的日志里；不一致会标 warning，
此时需要的内核模块只能编进固件。代价是构建更久（要编所有 kmod）。

## 触发方式

仓库 → Actions → `Build 360T7 Firmware` → Run workflow（可改版本号、关闭 kmod 对齐）。
推送到 `main` 也会自动触发；只改文档时在提交信息里加 `[skip ci]`。

构建完成后在 run 页面底部下载 `360t7-ubootmod-<版本>` artifact，内含：

| 文件 | 用途 |
|---|---|
| `*-qihoo_360t7-ubootmod-squashfs-factory.bin` | **恢复网页上传这个**（完整 UBI 镜像，全新安装） |
| `*-qihoo_360t7-ubootmod-squashfs-sysupgrade.bin` | 已经装了本固件后，系统内 `sysupgrade` 升级（保留配置） |
| `*-qihoo_360t7-ubootmod-initramfs-kernel.bin` | 调试用，恢复网页不吃它 |
| `sha256sums` | 校验 |

## 刷机步骤

1. **备份**（在当前系统里，不影响使用）：
   ```sh
   for i in 1 2 3 4 6 7; do cat /dev/mtd$i > /tmp/mtd$i.bin; done
   md5sum /tmp/mtd*.bin
   ```
   用 `scp -O` 拷到电脑。**`Factory`（无线校准数据）最重要**，多存几份。旧的 `/etc/config` 只作对照，不要导入新系统。
2. **核对** `factory.bin` 的 sha256。
3. **进恢复网页**：拔电 → 按住 reset 通电 5–10 秒 → 电脑网线接 LAN 口，网卡设 `192.168.1.2/24` → 打开 `192.168.1.1`。
4. 上传 `*-squashfs-factory.bin`，等待写入并重启。
5. 电脑网卡改回自动获取（DHCP），打开新系统 `192.168.1.1`，先设置 root 密码，再逐项配置。
6. 出问题：回到第 3 步，重刷本固件，或者刷官方
   `immortalwrt-23.05.7-mediatek-filogic-qihoo_360t7-ubootmod-squashfs-factory.bin` 兜底。

## 红线

- **永远不要刷 BL2 / FIP / preloader**，也不要用官方 `qihoo_360t7` 的 `*.itb`、`bl31-uboot.fip`、`preloader.bin`。
- 从别的固件**首次**装本固件，用恢复网页刷 `factory.bin`，不要在旧系统里 `sysupgrade -F`（板名和布局都不同）。

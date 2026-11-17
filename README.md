# OpenWrt NN6000 v2 — BTF kernel build

为 Link NN6000 v2 (ipq60xx, IPQ6000/IPQ6018) 编译带 **BTF** 的官方 OpenWrt 内核，
用于运行 CO-RE eBPF 程序（dae 等），同时保持与现有 rootfs 里 kmod 的 vermagic 兼容。

## 为什么能只刷内核

- 该机 eMMC 分区里内核与 rootfs 物理分离、按分区名独立写入：
  - `0:HLOS` = `/dev/mmcblk0p10` (12M, sector 9506–34081) → 内核 FIT image
  - `rootfs` = `/dev/mmcblk0p11` (2G, sector 34082–) → 系统 + kmod
  - 官方 `platform.sh` 用 `CI_KERNPART="0:HLOS"` + `emmc_do_upgrade` 按分区名写，
    `KERNEL_SIZE` 只决定 factory.bin 拼接时的 padding，**不改变分区布局**。
- CI 锁定到 `7424e2ca30`（r38384），与机器原厂内核同一 commit/基线。

## 关键：内核必须用 gzip FIT，不能用 LZMA

- 该机 U-Boot（高通 APPSBL）**只认官方的 gzip FIT**，不解压 LZMA FIT。
  实测把内核改成 `Device/FitImageLzma` 后，无论只刷内核还是整包 factory.bin，
  开机都直接掉进 U-Boot 命令行。只有 MikroTik 那类设备配了
  `mikrotik-lzma-loader` 自解压引导才能用 LZMA，本机没有。
- BTF + eBPF 选项让 gzip 内核涨到约 6.6M > 官方 `KERNEL_SIZE=6144k`，
  所以 CI 会 patch `KERNEL_SIZE := 12288k`（p10 物理满容量 12M），
  让 gzip 镜像能通过大小检查并 pad 满整个分区。

## 如何触发编译

手动触发：GitHub 网页 → Actions → `Build NN6000 v2 kernel only (BTF, gzip)` → Run workflow。
- `openwrt_ref` 保持默认（`7424e2ca30`）。
- 想要整包：`build_image` 勾上；只要内核：保持默认 false。
- 当前 workflow 只配了手动触发，push 不会自动编译。

首次约 40–90 分钟（GitHub Actions 免费额度内，公网 runner 性能一般）。

编译完成后会自动：
1. 上传 Actions artifact（保留 90 天）
2. 创建一个 prerelease，镜像文件在 Release 附件里

## 产物说明

| 文件 | 用途 |
|---|---|
| `kernel-hlos.itb` | **只刷内核**，gzip FIT pad 到 12288k（满 p10），写入 `0:HLOS`，保留现有 rootfs/配置 |
| `*-factory.bin` | 完整镜像，U-Boot 整包刷（内核+rootfs 都换） |
| `*-sysupgrade.bin` | 完整镜像，系统内 `sysupgrade` 用 |
| `*-initramfs-uImage.itb` | 救援内核，U-Boot tftp 启动、不写盘 |
| `build.config` | 实际使用的 .config |

## 刷机

### 方案 A：只刷内核（推荐，保留现有系统）

⚠️ 刷内核有变砖风险，务必先备份。之前已从机器备份 `p10_hlos.bin` 到本地。

系统内能 SSH 时：
```sh
dd if=/dev/mmcblk0p10 of=/tmp/hlos-backup.bin
dd if=kernel-hlos.itb of=/dev/mmcblk0p10 bs=1M conv=fsync
reboot
```

或 U-Boot 下 tftp 写入 `0:HLOS` 分区（需串口）。

### 方案 B：整包 factory.bin

U-Boot 直接刷写（与最初刷 snapshot 的方式相同）。

### 验证 BTF

```sh
ls -l /sys/kernel/btf/vmlinux
bpftool btf dump file /sys/kernel/btf/vmlinux format c | head
modinfo tun | grep vermagic   # 应与现有 kmod 一致
```

## 回滚

内核刷坏时用 U-Boot 恢复 `0:HLOS`，写入本地备份的 `p10_hlos.bin`。

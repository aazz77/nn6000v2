# OpenWrt NN6000 v2 — BTF kernel build

为 Link NN6000 v2 (ipq60xx, IPQ6000/IPQ6018) 编译带 **BTF** 的官方 OpenWrt 内核，
用于运行 CO-RE eBPF 程序（dae 等），同时保持与现有 rootfs 里 kmod 的 vermagic 兼容。

## 为什么能只刷内核

- 该机 eMMC 分区里内核与 rootfs 物理分离：
  - `0:HLOS` = `/dev/mmcblk0p10` (12M) → 内核 FIT image
  - `rootfs` = `/dev/mmcblk0p11` (2G) → 系统 + kmod
- BTF/DEBUG_INFO 是调试信息选项，**不改变内核 vermagic 与符号 CRC**，
  因此同一 commit、默认 config 下只加 BTF，新内核与现有 kmod 兼容。
- CI 锁定到 `7424e2ca30`（r38384），即当前机器内核的基础 commit。

## 如何触发编译

两种方式：
1. **push 到 main**（推荐，符合日常习惯）：任何 push 都会自动触发编译。
2. 手动：GitHub 网页 → Actions → `Build OpenWrt NN6000 v2 (BTF kernel)` → Run workflow，
   `openwrt_ref` 保持默认即可。

首次约 40–90 分钟（GitHub Actions 免费额度内，公网 runner 性能一般）。

编译完成后会自动：
1. 上传 Actions artifact（保留 90 天）
2. 创建一个 prerelease，镜像文件在 Release 附件里

## 产物说明

| 文件 | 用途 |
|---|---|
| `kernel-hlos.itb` | **只刷内核**，6144k，写入 `0:HLOS`，保留现有 rootfs/LuCI/配置 |
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

# UniXNU-xun

Redmi Pad SE（xun，骁龙 680 / SM6225，eMMC）的 UniXNU 移植骨架。

参考已经能进 iPadOS 17.7.11 主屏幕的小新 Pad 2022：

- 设备仓库：[xVanTuring/UniXNU-tb128fu](https://github.com/xVanTuring/UniXNU-tb128fu)
- 固件：[xVanTuring/uefi-tb128fu](https://github.com/xVanTuring/uefi-tb128fu)
- 文档：[unixnu.com/zh/devices/tb128fu](https://unixnu.com/zh/devices/tb128fu)

**这不是能刷的成品。** SoC 相同，板级不同。直接把 tb128fu 的镜像刷进 xun 会黑屏，存储路径也对不上。

## 能直接抄的

| 项 | 两边 |
| --- | --- |
| SoC | SM6225 / bengal，4×A73 + 4×A53 |
| GPU | Adreno 610，无 GMU |
| 存储介质 | 用户确认两边都是 eMMC（见下面和官方文档的冲突） |
| 启动链 | 纯 Mu。原厂 ABL 没有可拆的厂商 DXE，显示先用 ABL 留下的 splash framebuffer |
| 内核参数 | EL1 虚拟定时器 INTID 19、SGI 收不到所以 IPI 用软件挂起的 SPI、`sheng-pmgr=2` 这类 SM6225 特例 |

这些写在 tb128fu 文档的「说明」里，是 SoC 行为，不是联想屏的行为，xun 大概率同样适用。

## 不能抄的

| 项 | tb128fu（小新 Pad 2022） | xun（Redmi Pad SE） |
| --- | --- | --- |
| 屏幕 | 2000×1200，固件 framebuffer | 1920×1200 IPS。面板有三套：`m84_35_03_0b`、`m84_36_02_0a`、`m84_42_03_0c`（BOE / 华星 / 天马） |
| 触控 | SPI 上的 Himax HX83102E，内核下固件 | I2C 上的敦泰 FT8203（BOE、华星两套固件）或诺瓦天马。**不是 Himax，不能用 TouchProbe.cpp** |
| 存储驱动 | 仓库写的是 NativeUfs / `native-ufs-root=1`，链路 HS-G3 | `fstab.emmc`：`/dev/block/by-name/*` 走 emmc，SD 卡节点是 `4784000.sdhci`。启动介质是 eMMC，不是 UFS LUN |
| boot 分区 | header v1，FD 在 `0x5FC00000`，要 `fastboot flash boot_a`（`fastboot boot` 缓冲区太小） | header **v4**，`boot` 128MB，另有 `init_boot`、`vendor_boot`。payload 怎么塞进 v4 还没验证 |
| 内存 / GPT | `userdata` 从平板 GPT 读出：`first_lba = 0x222f31` | **还没从真机读**。不能沿用 tb128fu 的 LBA |

官方 tb128fu 状态表写「UFS 上的根和 Data 卷」，README 也写 UFS HS-G3。你说两台都是 eMMC。SM6225 的控制器两边都有，**介质以 fstab 为准**：xun 的系统分区明确标了 `emmc`。tb128fu 那边要么机器实际是 UFS，要么文档把 SDHCI/UFS 主机叫混了。xun 这一侧先按 eMMC 做，不要打开 `native-ufs-root`。

## 建议的阶段

1. **只拿菜单和 splash。** 用 tb128fu 的 Mu 树加 `xunPkg`，DTB 换成 xun 的 Khaje/bengal 树，`SimpleFbDxe` 吃 ABL 留下的 `cont_splash_region`。先确认 UEFI 菜单出现。
2. **eMMC 读块。** 不要链 `Tb128fuUfs --sm6225`。需要一份走 `4784000.sdhci` 的块设备，或者先用 ramdisk 把根放进内存，存储往后放。
3. **framebuffer 出画面。** 分辨率按 1920×1200，不要写死 1200×2000。
4. **触控最后做。** 驱动得换成敦泰 FT8203（或诺瓦），协议和 Himax SPI 完全不同。
5. **GPU** 可以后接 Adreno 610 那条，和 tb128fu 同一颗，但 SMMU 窗口和 zap 固件要用 xun 自己的。

## 还缺的真机数据

没这些就不要填 `board.toml` 里的 LBA 和内存银行：

- `fastboot getvar partition-size:boot`、`partition-size:userdata`
- 一份 `sgdisk -p /dev/block/mmcblk0` 或 Android 里的 `blockdev --getsize64`
- `/proc/device-tree` 里的 `/memory`、`/reserved-memory/cont_splash_region`
- 屏幕是哪一套面板（设置里或 `dmesg` 里的 panel name）
- 触控是 FT8203 还是诺瓦（`dmesg | grep -E 'fts_|nvt'`）

## 本目录

| 文件 | 内容 |
| --- | --- |
| `docs/DELTA.md` | 和 tb128fu 的差异，给后面改 dsc/fdf 用 |
| `configs/xun_defconfig` | 从 `tb128fu_defconfig` 起的占位，**还没改存储驱动** |
| `board.toml.skel` | 骨架。LBA、内存、分辨率标了 TODO |

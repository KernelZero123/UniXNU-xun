# xun 真机读数（2026-10-31）

来源：用户在 Redmi Pad SE 上跑的 fastboot / adb。没有 root，`blockdev` 被拒绝。

## 启动槽和分区

- `current-slot: a`
- `partition-size:boot = 0x8000000`（128 MiB）
- `boot_a` / `boot_b` = `mmcblk0p85` / `p86`，`/proc/partitions` 里各 131072 个 1 KiB 块，正好 128 MiB
- `partition-size:userdata = 0x197BF00000`（109453508608 字节，104383 MiB）
- `userdata` = `mmcblk0p91`，同样是 106888192 个 1 KiB 块
- 整盘 `mmcblk0` = 122138624 KiB（约 119276 MiB）
- 块设备是 `mmcblk0`，不是 `sda`。存储介质是 eMMC

userdata 的起始 LBA 还没有。`/proc/partitions` 不带起始扇区，`blockdev --getsize64` 要 root。`board.toml` 里不能填 `first_lba`。

## 内存

`/proc/device-tree/memory/reg` 三段，大端：

| 基址 | 大小 | 结束 |
| --- | --- | --- |
| `0x40000000` | `0x3CC00000`（972 MiB） | `0x7CC00000` |
| `0x80000000` | `0x40000000`（1024 MiB） | `0xC0000000` |
| `0xC0000000` | `0x100000000`（4096 MiB） | `0x1C0000000` |

内核可见约 6092 MiB。`0x7CC00000` 到 `0x80000000` 是洞，不要合成一整段 `0x40000000` 起、长度 `0x180000000`。tb128fu 骨架里的 `dram_size = 0x180000000` 不能用。

## splash

没有 `cont_splash_region`。节点名是 `splash_region@5c000000`。

相邻保留区：

- `splash_region@5c000000`
- `dfps_data_region@5cf00000`
- `ramoops@5D000000`
- `removed_region@60000000`

splash 的 `reg` 没读到。如果它紧挨到 `0x5CF00000`，大小上限是 `0xF00000`（15 MiB），这只是上限，不是测量值。

tb128fu 的 FD 放在 `0x5FC00000`。xun 上 `ramoops` 从 `0x5D000000` 开始，`removed_region` 在 `0x60000000`。`0x5FC00000` 落在这两段之间，ramoops 有多大还不知道，所以 FD 基址不能照抄。

## 这块机器的屏和触控

内核命令行：

```
msm_drm.dsi_display0=qcom,mdss_dsi_panel_m84_42_03_0c_fhdp_video:
focaltech_ts_spi.tpsensor=focaltech_ts_ab
```

面板是三套里的 `m84_42_03_0c`，video mode，FHD+。

触控：

- 诺瓦驱动起来了，但 `tpsensor` 不是它，日志是 `Unknown Touch`
- 敦泰：`TP info: [Vendor]HUAXING [IC]ft8203`
- `chip id = 0x8203`
- 总线是 SPI：`4a80000.spi` / `spi0.0`，不是 I2C
- `display x(0 11999) y(0 19199)`，也就是 1200×1920（坐标按 10 倍记）
- irq gpio 478，reset gpio 484

所以这台是华星面板 + 敦泰 FT8203 SPI。不是诺瓦，也不是 tb128fu 的 Himax HX83102E。固件文件应对 `focaltech_ts_fw_huaxing_ft8203.bin`。

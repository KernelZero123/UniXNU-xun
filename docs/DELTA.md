# xun 相对 tb128fu 必须改的点

## 固件包

tb128fu 的平台文件：

- `Platforms/Lenovo/tb128fuPkg/tb128fu.dsc`
- `Platforms/Lenovo/tb128fuPkg/tb128fu.fdf`
- `Platforms/Lenovo/tb128fuPkg/Include/DXE.inc`
- `Platforms/Lenovo/tb128fuPkg/Library/MemoryMapLib`
- `Resources/Configs/tb128fu.toml`（FD `0x5FC00000`，3 MiB，header v1）
- `Resources/DTBs/tb128fu.dtb`

xun 建议新建 `Platforms/Xiaomi/xunPkg/`，不要改 Lenovo 那份。FD 基址和 splash 区域要从 xun 的 reserved-memory 重新量，**不要默认 0x5FC00000 / 0x5C000000**。

## 启动镜像

xun `BoardConfig.mk`：

- `BOARD_BOOT_HEADER_VERSION := 4`
- `BOARD_BOOTIMAGE_PARTITION_SIZE := 0x8000000`（128 MiB）
- 还有 `init_boot`、`vendor_boot`

tb128fu 用 header v1，把 72 MiB payload 刷进 `boot_a`。xun 的 boot 有 128 MiB，装得下，但 header v4 的 boot 里通常没有 ramdisk（ramdisk 在 `init_boot` / `vendor_boot`）。Mu 的 `header_version = 1` 在这台 ABL 上能不能被吃掉，要试。试的顺序：

1. 小镜像 `fastboot boot`（tb128fu 上这个缓冲区太小，xun 不一定）
2. 不行再 `fastboot flash boot`，并且先备份 `boot_a` 和 `boot_b`

## 存储

xun `fstab.emmc` 的系统分区是 `emmc` + `slotselect`。内核参数里删掉：

- `native-ufs-root=1`
- `native-ufs-diagnostics=2`
- `build-arm-driver.py Tb128fuUfs --sm6225`

在 eMMC 驱动能读 GPT 之前，根放 ramdisk。`[userdata]` 的 `first_lba` 留空。

## 显示

tb128fu 文档写死 1200×2000。这台机器触控坐标是 1200×1920。`SimpleFbDxe` 读 `splash_region@5c000000` 的真实宽高，不要写死 tb128fu 的 1200×2000。

## 触控

这台是敦泰 FT8203，华星，SPI `4a80000.spi`。不要编译 `TouchProbe.cpp` / `HimaxProtocol`。固件文件是：

- `focaltech_ts_fw_boe_ft8203.bin`
- `focaltech_ts_fw_huaxing_ft8203.bin`
- `novatek_ts_fw_tianma.bin`

先确认手上这台是哪颗 IC，再写驱动。主屏幕可以先用 `offline-hid-home=1` 的虚拟 Home 键顶着。

## 面板

三选一，不能写死：

- `m84_35_03_0b`
- `m84_36_02_0a`
- `m84_42_03_0c`

第一阶段不初始化面板，只用 ABL 已经点亮的 framebuffer。

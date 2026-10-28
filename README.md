# AGNOS for OnePlus 6T (fajita / SDM845 v2.1)

把 comma.ai 的 AGNOS 内核移植到一加 6T。

## 硬件
- SoC: SDM845 v2.1, msm-id `<0x141 0x20001>`
- 面板: 1080×2340 AMOLED, `samsung_s6e3fc2x01_cmd` / `samsung_dsc_cmd`
- 触摸: Synaptics S3320, i2c12 @0x20, gpio reset=99 irq=125 1v8=88
- 相机: IMX519(主) + IMX376K(广) + IMX371(前)
- 音频: Tavil + 4×WSA881x
- PMIC: PM8998 + PMI8998 + 2×SMB1355

## 目录
- `dts/sdm845-oneplus6t.dts` — 完整设备树（26,897 行，从真机 DTB 反编译 + dtbo merge）
- `defconfig/fajita_defconfig` — 基于 tici_defconfig 修改
- `.github/workflows/build-kernel.yml` — CI 编译流水线

## 关键改动（vs tici_defconfig）
| 配置 | 改动 | 原因 |
|---|---|---|
| `CONFIG_MSM_CAMERA` | not set → `y` | camerad 需要 camera_v2 V4L2 |
| `CONFIG_SND_SOC_WCD934X` | → `y` | 音频编进内核，避免 vermagic 失配 |
| `CONFIG_SND_SOC_WSA881X` | → `y` | 同上 |
| `CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE_NAMES` | → `"qcom/sdm845-oneplus6t"` | DTB 内嵌 |

## 设备树来源
从 6T 备份的 `boot_a.img` 拆出：
- gzip(Image) 12,171,374 字节
- DTB 区 1,239,226 字节（3 个 base SoC DTB，选中 v2.1）
再 merge `dtbo_a.img` 的 24 个 overlay 中的 `dtbo_01`，得到完整设备树。

已验证：dtc 往返编译行数一致（26,897 行），关键节点（panel/touch/tavil/camera）全部保留。

# 6T AGNOS 构建方案（Lead 已本地验证的部分）

## 已就绪的输入文件

| 文件 | 位置 | 状态 |
|---|---|---|
| 6T 完整设备树 | `agnos-port/ready-to-build/dts/sdm845-oneplus6t.dts` | ✅ dtc 编译验证通过（574,846 字节） |
| 原始 merged DTB | `agnos-port/hw-extract/boot_dtb_merged.dtb` | 574,913 字节 |
| 内核 config 参考 | `agnos-port/hw-extract/kernel-config.txt` | 4610 项 |
| 硬件清单 | `agnos-port/hw-extract/HW-INVENTORY.md` | 760 行 |
| 关键节点表 | `agnos-port/hw-extract/dts-key-nodes.md` | 776 行 |

## 关键决策：不走上游 include 链

agnos-kernel 的 DTS 是 `comma_tizi.dts → sda845-v2.1.dtsi → sdm845-v2.1.dtsi → sdm845.dtsi` 的 include 链。
6T 的 DTS 是**从真机 DTB 反编译出来的完整单文件**（26,897 行，无 include）。

**做法：把 6T 的 dts 作为独立 board dts 直接放进内核，不拆成 dtsi。**
理由：
- 已经验证过它能独立编译成 DTB
- 所有节点、phandle、GPIO 号都是真机实测值，改动越少越安全
- 避免 include 链导致的 phandle 冲突

## 编译流程

```bash
# 1. 取内核源码
git clone --depth 1 https://github.com/commaai/agnos-kernel-sdm845.git
cd agnos-kernel-sdm845

# 2. 放入 6T 设备树
cp /path/sdm845-oneplus6t.dts arch/arm64/boot/dts/qcom/

# 3. 注册到 Makefile
echo 'dtb-$(CONFIG_ARCH_QCOM) += sdm845-oneplus6t.dtb' >> arch/arm64/boot/dts/qcom/Makefile

# 4. 复制 tici_defconfig 并按 6T 修改
cp arch/arm64/configs/tici_defconfig arch/arm64/configs/fajita_defconfig
# 关键改动见 DEFCONFIG-CHANGES（等 kernel-analyst 交付）

# 5. 交叉编译
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
make fajita_defconfig
make -j$(nproc) Image.gz-dtb

# 6. 产物
# arch/arm64/boot/Image.gz-dtb
```

## 打包 boot.img

```bash
# 需要 mkbootimg（android-tools-mkbootimg 或 pip install mkbootimg）
mkbootimg \
  --kernel arch/arm64/boot/Image.gz-dtb \
  --ramdisk <AGNOS ramdisk> \
  --base 0x00000000 \
  --kernel_offset 0x00008000 \
  --ramdisk_offset 0x01000000 \
  --tags_offset 0x00000100 \
  --pagesize 4096 \
  --header_version 1 \
  --cmdline "androidboot.hardware=qcom androidboot.console=ttyMSM0 video=vfb:640x400,bpp=32,memsize=3072000 msm_rtb.filter=0x237 ehci-hcd.park=3 lpm_levels.sleep_disabled=1 service_locator.enable=1 swiotlb=2048 androidboot.configfs=true loop.max_part=7 androidboot.usbcontroller=a600000.dwc3 rootwait ro init=/init" \
  -o boot.img
```

## 刷入

```bash
adb reboot bootloader
fastboot --disable-verity --disable-verification flash vbmeta_a vbmeta_empty.img
fastboot flash boot_a boot.img
fastboot reboot
```

## 本地已验证
- ✅ 6T.dts 可编译成 DTB（574,846 字节）
- ✅ 反编译回 dts 行数一致（26,897 行）
- ✅ 关键节点完整（panel/touch/tavil/camera）

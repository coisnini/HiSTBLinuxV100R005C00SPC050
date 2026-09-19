# HDMI 控制台支持（hifb + fbcon）—— 改动说明

这一目录是给 `HiSTBLinuxV100R005C00SPC050` 打的补丁集，目标：让 Hi3798MV300
板子的内核启动日志（fbcon 跑码）能显示在 HDMI 上，并且显示器上是黑底白字。

板子：Hi3798MV300（Hi3798MV3DMW），eMMC，AArch64，Ubuntu 22.04 rootfs，
内核 `Linux 4.4.35-hinas-console64`。

## 三个补丁

按顺序 `git apply`：

| 顺序 | 文件 | 改哪 | 作用 |
| --- | --- | --- | --- |
| 1 | `0001-fbdev-Kconfig-cfb-default-y.patch` | `source/kernel/linux-4.4.y/drivers/video/fbdev/Kconfig` | 把 `FB_CFB_FILLRECT` / `FB_CFB_COPYAREA` / `FB_CFB_IMAGEBLIT` 的 `default n` 改成 `default y`。这三个不打开，fbcon 画不出字，屏幕全黑 |
| 2 | `0002-hifb-enable-console.patch` | `source/msp/drv/hifb/src/drv_hifb_osr.c` | 让海思 hifb 图形层真正支持 framebuffer console（注册 fb_ops 里的绘图回调、允许 `fb0` 被 fbcon 接管） |
| 3 | `0003-hifb-opaque-alpha.patch` | `source/msp/drv/hifb/src/drv_hifb_osr.c` | 把图层整体透明度 `u8Alpha0/Alpha1/GlobalAlpha` 从 `HIFB_ALPHA_TRANSPARENT` 改成 `HIFB_ALPHA_OPAQUE`，并让 ARGB1555 调色板值恒置 bit15。缺这一刀的表现是**能跑码但屏上看不见字**（整屏发绿或全黑） |

第 2 和第 3 个补丁改同一个文件，`git status` 里只会看到一次
`M source/msp/drv/hifb/src/drv_hifb_osr.c`，这是正常的。打完务必用两道 grep 确认：

```bash
grep -n 0x8000 source/msp/drv/hifb/src/drv_hifb_osr.c              # 补丁 3 第一处
grep -n HIFB_ALPHA_OPAQUE source/msp/drv/hifb/src/drv_hifb_osr.c   # 补丁 3 第二处（决定性）
```

## 两个配置文件

| 文件 | 要放到的位置 |
| --- | --- |
| `hi3798mv3dmw_hi3798mv300_console64_cfg.mak` | `configs/hi3798mv300/`（SDK 构建配置，arm64 + VRAM 16200 + console 支持） |
| `hi3798mv300_console64_defconfig` | `source/kernel/linux-4.4.y/arch/arm64/configs/`（内核 defconfig，含 `CONFIG_FRAMEBUFFER_CONSOLE=y`、`CONFIG_DEVTMPFS_MOUNT=y`、`# CONFIG_ANDROID_PARANOID_NETWORK is not set`） |

## 编译

```bash
source ./env.sh
make -j4 linux SDK_CFGFILE=configs/hi3798mv300/hi3798mv3dmw_hi3798mv300_console64_cfg.mak
```

产物：`out/hi3798mv300/hi3798mv3dmw_console64/obj64/source/kernel/linux-4.4.y/arch/arm64/boot/Image`

- 尺寸必须是 **20,316,160** 字节；
- 末尾那句 `Error: Invalid platform ... hi3798mv300`（ATF 不支持 mv300）**无害**，
  内核在它之前已经编完（日志里会出现 `Image arch/arm64/boot/uImage is ready`）。

## 校验

```bash
grep -c hisilicon_fephy_s28v112_fix System.map   # 必须 >= 1
grep -c hisilicon_fephy300_fix System.map        # 必须 == 0
```

注意：`Image` 的字节不是可比特复现的（同一台机器编两次也会差几 MB，
`.rodata` 里的 tracepoint 指针数组顺序会变），所以**不要拿 Image 的 MD5 去比对**，
要比的是尺寸、`.config`、以及 System.map 里的 PHY 符号。

`Image` 打进 FIP 之后，除内核载荷和 uImage 头 CRC 之外，其余字节应当与原厂 p6 完全一致。
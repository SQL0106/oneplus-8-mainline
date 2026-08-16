# oneplus-8-mainline

OnePlus 8（`oneplus-instantnoodle`，高通 SM8250）主线 Linux / postmarketOS 构建仓库。

通过 GitHub Actions 自动构建可刷机镜像（boot.img + rootfs），产出物在每次构建的
Artifacts 中下载。

## 项目结构

```
packages/
├── device-oneplus-instantnoodle/    # 设备包（deviceinfo、dtbo、内核 cmdline）
├── linux-oneplus-instantnoodle/     # 主线内核包（Xo666 v6.16.7，op8_defconfig）
├── firmware-oneplus-instantnoodle/  # 固件包（adsp/slpi/cdsp/venus/a650/ath11k/qca）
└── alsa-oneplus-instantnoodle/      # ALSA UCM2 音频配置
```

- 内核源码：<https://github.com/Xo666/mainline-instantnoodle>（分支 `6.16.7`，commit `408762b`）
- 固件来源：<https://github.com/Xo666/linux-oneplus-instantnoodle>
  （蓝牙固件取自同平台 OnePlus 8 Pro 固件包，QCA6390 芯片相同）
- pmOS 参考：`device-oneplus-instantnoodle` 暂未合入上游 pmaports，本仓库以 overlay 方式注入 CI

## 触发构建

- **手动**：Actions → Build → Run workflow（可选填 `ui`：`console` / `phosh` / `sxmo`，默认 headless `console`）
- **自动**：push 到 `main`（README 变更除外）

构建完成后下载 Artifacts 里的 `postmarketos-oneplus-instantnoodle`，内含：
`boot.img`、`<device>.img`（rootfs，稀疏格式）、`dtbo.img`。

## 刷机（fastboot）

> ⚠️ **会清空 Android 的 super 动态分区。务必先备份 super 分区再刷。**

1. 备份原厂 super 分区（Android 系统下，一次即可）：
   ```sh
   adb root
   adb pull /dev/block/by-name/super super.img
   ```
2. 关机，按住 `音量-` + `音量+` + `电源` 进入 fastboot（bootloader 需已解锁）。
3. 刷入：
   ```sh
   fastboot flash boot   boot.img
   fastboot flash super  oneplus-instantnoodle.img   # 稀疏镜像，fastboot 直接支持
   fastboot flash dtbo   dtbo.img
   fastboot reboot
   ```

headless 模式下开机后设备处于 172.16.42.1/24，通过 USB 网络用 SSH 登录：

```sh
ip link set <usb网卡> up
ip addr add 172.16.42.2/24 dev <usb网卡>
ssh user@172.16.42.1   # 密码随机生成，在构建日志中
```

### 恢复 Android

```sh
# super.img 为步骤 1 备份的原厂分区
img2simg super.img super-s.img
fastboot flash super super-s.img
fastboot reboot
```

## 功能状态（Xo666 内核 6.16.7）

| 功能 | 状态 |
| --- | --- |
| 启动 / USB 网络 / SSH | 可用 |
| 屏幕 / 触摸 | 可用 |
| WiFi / 蓝牙 | 可用 |
| 音频（WCD938x + TFA9874×2） | 可用 |
| 电池电量（bq27411） | 可用 |
| 充电 | 仅 5W（Warp 30W 无驱动） |
| 前置相机（imx471） | 部分可用 |
| NFC | 可用 |
| 电话 / SMS / 移动数据 | 不支持（modem 未启用） |
| GPS / 传感器 / 马达 | 未测试 |
| 插电自动开机 | 不支持（pmOS 无此机制，关机插电需按电源键） |

## 已知坑

- pmbootstrap 的 `mkinitfs` 曾报 `found: []`：原因是内核包未正确安装 kernel.release；
  本仓库的 `linux-oneplus-instantnoodle` 按 pmOS 标准流程安装，已规避。
- 内核配置基于 Xo666 的 `op8_defconfig`，CI 中额外关闭 `DEBUG_INFO` 以减小体积、加快编译。
- 首次构建需下载内核源码（约 250MB）并编译约 40-60 分钟；ccache 已接入，重新构建会快很多。

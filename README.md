# oscar (RMX3478 / Realme Q5) ReSukiSU + SUSFS 内核 GitHub Actions 构建

为 **Realme Q5 / 9 5G (oscar, RMX3478)** 定制的 ReSukiSU + SUSFS 隐藏 SELinux 内核自动构建。
平台 SM6375 (holi)，内核 5.4 **QGKI**，基于 LineageOS `android_kernel_realme_sm6375`（默认 lineage-23.2 分支，对应你已刷的 LineageOS 23.2）。

## 原理

```
LineageOS 内核源码(lineage-23.2)
   + ReSukiSU (builtin 集成, SukiSU-Ultra 重构 fork, 自带 selinux_hide)
   + SUSFS v2.2.0 (fs/ 层 5.4 移植 patch, 隐藏/伪造文件系统视图)
   + AOSP clang (r510928) 编译 vendor/holi-qgki_defconfig
   + 从 LineageOS nightly zip 提取原版 boot.img 的 ramdisk
   = 打包 new_boot.img (新内核 + 原 ramdisk + 原 cmdline)
```

产物 `new_boot.img` 就是 **带 root + 隐藏 SELinux 能力的新 boot**，fastboot 刷入即可。

## 使用方法

### 1. 建仓库

在 GitHub 新建一个仓库（如 `oscar-sukisu-build`），然后把本目录内容传上去：

```bash
cd G:\AIWORK\abl解密\sukisu-oscar
git init
git add .
git commit -m "oscar sukisu build"
git branch -M main
git remote add origin https://github.com/<你的用户名>/oscar-sukisu-build.git
git push -u origin main
```

（也可以在网页端直接上传文件）

### 2. 触发构建

1. 仓库 → **Actions** 页面
2. 左侧选 **Build SukiSU Kernel for oscar**
3. 点 **Run workflow**
4. 填写：
   - `kernel_branch`：`lineage-23.2`（默认，不用改）
   - `los_version`：`23.2`（默认）
   - `los_date`：**必须填你手机当前系统的 nightly 日期**，你刷的是 `20260817`，保持默认即可
   - `custom_version`：可选，自定义版本名
5. Run

### 3. 下载产物

构建结束（约 40-90 分钟）后，在 workflow 运行页底部 **Artifacts** 下载：
- `new_boot.img` ← **这个就是内核 + root 的 boot 镜像**

### 4. 刷入（Windows）

```bat
cd /d E:\ruanjian\工具房\windows\platform-tools-windows\platform-tools
adb reboot bootloader
fastboot flash boot_a new_boot.img
fastboot flash boot_b new_boot.img
fastboot reboot
```

### 5. 装管理器

- 下载 [ReSukiSU Manager](https://github.com/ReSukiSU/ReSukiSU/releases)（也兼容 SukiSU-Ultra / KernelSU Manager）
- 安装打开 → 显示已授权即成功

## 隐藏 SELinux（本项目核心）

ReSukiSU 自带 **selinux_hide**（5.4 兼容实现），管理器里勾选后：
- `getenforce` 冻结在 `Enforcing`，不会变 Permissive
- su 进程的 SELinux context 伪装为 `u:r:su:s0`（非 `magisk`/`su` 域），规避检测

SUSFS 9 项能力（内核侧全部开启，由 susfs4ksu 工具按需启用）：

| 功能 | 用途 |
|---|---|
| SUS_PATH | 隐藏可疑路径/文件（如 magisk 目录） |
| SUS_MOUNT | 伪造挂载表，隐藏挂载点 |
| SUS_KSTAT | 伪造 stat/statfs 结果 |
| OPEN_REDIRECT | 打开隐藏路径时重定向到伪装路径 |
| SUS_MAP | 隐藏 /proc/self/maps 中的内核映射 |
| SPOOF_CMDLINE_OR_BOOTCONFIG | 伪造 /proc/cmdline |
| SPOOF_UNAME | 伪造 uname |
| ENABLE_LOG / HIDE_SYMBOLS | 日志与符号隐藏 |

## 验证 root

- 设置 → 关于手机 → 内核版本：应显示 `5.4.xxx-...`（含你自定义版本名则有显示）
- 终端或 root 管理器执行 `su` 验证

## ⚠️ 重要提醒

1. **los_date 必须与你手机当前系统一致**——ramdisk 不匹配可能导致启动异常
2. **刷错 boot = 卡 logo**，不丢数据，救回方法：
   ```bat
   fastboot flash boot_a boot.img   :: 用 F:\lineage\boot.img 原版
   fastboot flash boot_b boot.img
   ```
3. 每次 **升级 LineageOS（dirty flash 或 OTA）后 root 会丢**，需重新下载对应日期的构建并重刷（把 `los_date` 改成新 nightly 日期再跑一次 workflow）
4. 首次构建全量编译较慢（40-90 分钟），GitHub Actions 免费额度 2000 分钟/月足够
5. 基带/指纹等硬件功能在刷 boot 时不受影响（dtbo 未动）；若遇到显示/触摸异常，把 `F:\lineage\dtbo.img` 双槽刷回

## 缓存机制（已内置）

构建工作流带四层缓存，**第二次开始明显提速**：

| 缓存项 | key | 效果 |
|---|---|---|
| 内核源码 | `kernel-oscar-<分支>` | 首次下载后永久复用（~1GB） |
| AOSP Clang | `clang-oscar-r510928-v1` | 首次下载后永久复用（~2GB） |
| 原版 boot.img | `boot-oscar-<los_date>` | 同日期只下载一次（160MB） |
| ccache 编译缓存 | `ccache-oscar-<分支>-<run>` | **每次成功构建后更新，二次编译从 60 分钟降到 10-20 分钟** |

- 查看/清理：仓库 → Actions → 左侧 **Cache**（或 Settings → Actions → Cache）
- 缓存总量受仓库配额限制（免费 10GB），GitHub 自动淘汰最旧条目，无需手动管理
- 升级 nightly 后（改 `los_date`）：boot.img 缓存自动失效重建，ccache 增量命中（内核 95% 相同，仍然很快）

---

## 疑难排解

| 现象 | 原因 | 处理 |
|---|---|---|
| 编译失败：`CONFIG_KSU` 未找到 | ReSukiSU 集成失败或 Kconfig 符号名不同 | 看 workflow 日志 "检查 ReSukiSU 源码是否集成" 一步 |
| 编译失败：SUSFS patch 拒绝应用 | 内核分支不是 lineage-23.2（源码版本不符） | 确认 kernel_branch；若 LineageOS 更新了内核源码，需要重新移植 patch |
| `clang` clone 超时 | googlesource 网络抖动 | 重新 Run workflow（幂等） |
| nightly 下载失败 | 日期填错或版本前缀不符 | 核对 `los_date` 与 `los_version`，去 lineageos 官网确认该日期存在 |
| 刷入后卡 logo | ramdisk 与系统版本不匹配 | 用原版 boot.img 救回，确认 `los_date` 正确后重试 |
| 刷入后 WiFi/蓝牙/音频全挂，dmesg 报 `disagrees about version of symbol module_layout` | 官方内核 `CONFIG_CFI_CLANG=y`（clang21+LTO+LLD），`struct module` 多一个 `cfi_check` 字段 → module_layout CRC 与自编译内核不同，MODVERSIONS 校验拒绝所有 vendor 预编译模块 | 已内置两步修复：① `60_skip_modversions_crc.patch` 跳过 CRC 校验（vermagic 仍校验）② `70_add_cfi_slowpath.patch` 导出假 `__cfi_slowpath`（官方 CFI 模块引用该符号，不开 CFI 会报 unknown symbol）。两条链缺一不可 |
| 想用 KernelSU 而非 ReSukiSU | 方案不同 | 集成脚本换成 KernelSU 的 `kernel/setup.sh`（非 GKI 教程见 kernelsu.org） |

## 参考

- ReSukiSU: https://github.com/ReSukiSU/ReSukiSU
- SUSFS: https://gitlab.com/simonpunk/susfs4ksu
- LineageOS oscar 内核: https://github.com/LineageOS/android_kernel_realme_sm6375
- oscar 设备信息: https://wiki.lineageos.org/devices/oscar/variant3/
- 构建模板参考: https://github.com/ShirkNeko/GKI_KernelSU_SUSFS（纯 GKI，本仓库为其 QGKI 5.4 定制版）

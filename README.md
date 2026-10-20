# cibuilder

为 **Oculus Quest 3** 构建 **ReSukiSU** 内核模块（`kernelsu.ko`）的 GitHub Actions 工作流。

## 怎么做

1. Fork 本仓库
2. 打开 **Actions** 标签，运行 **ReSukiSU LKM Build (Oculus Quest 3)** 工作流
3. 构建完成后，从运行页面下载 **kernelsu-ko** 产物

在设备上加载：

```sh
pm path com.resukisu.resukisu
cd /data/app/<...>/lib/arm64
./libksud.so insmod /data/local/tmp/kernelsu.ko
```

## 为什么这么做

Quest 3 运行的是 Meta 定制内核（`5.10.240-g9cb2f620a9a8`）。Meta 破坏了部分 GKI 兼容性，通用 GKI/DDK 构建出的模块无法加载，因此必须满足两个条件：

1. **对着 Quest 3 的真实内核源码编译**（`facebookincubator/oculus-linux-kernel`，`oculus-quest3-kernel-master` 分支）
2. **用设备上导出的完整配置**（`device_defconfig`）作为内核配置，否则编译出的模块 vermagic 与设备不一致，无法加载

工作流流程：

```
git clone oculus-linux-kernel (commit 8f32bdea)
  ├─ 用 device_defconfig 配置内核（禁用无法独立构建的驱动）
  ├─ 编译 vmlinux (thin LTO, clang 14)
  └─ off-tree 编译 ReSukiSU 模块 (CONFIG_KSU=m, make M=KernelSU/kernel)
```

编译出的 `kernelsu.ko` vermagic 为 `5.10.240-g9cb2f620a9a8 SMP preempt mod_unload modversions aarch64`，与设备完全一致；设备 config 未开启 `CONFIG_MODULE_SIG`，未签名模块可直接加载。

## 换到其他设备/系统

修改 `.github/workflows/build.yml` 和 `device_defconfig`：

| 位置 | 当前值（Quest 3） | 需要改成 |
|---|---|---|
| `device_defconfig` | 从本机 `/proc/config.gz` 导出 | 从目标设备导出 |
| workflow：内核源码 clone 的分支/commit | `oculus-quest3-kernel-master` / `8f32bdea` | 目标设备内核的分支和 commit |
| workflow：`CONFIG_LOCALVERSION` | `-g9cb2f620a9a8` | 目标设备的 kernel release（`uname -r`） |
| workflow：禁用驱动的 `sed` 行 | `QCA_CLD_WLAN`、`OCULUS`、`META`、`QCOM_KGSL` | 目标内核里无法独立编译的驱动 |
| workflow：clang 分支 | `llvm-r450784` / `clang-r450784c` | 与设备内核实际使用的编译器一致 |
| workflow：`com.resukisu.resukisu` | ReSukiSU 管理器包名 | 若使用其他管理器则对应更改 |

`CONFIG_LOCALVERSION` 必须与设备 `uname -r` 完全一致，否则 vermagic 不匹配、模块无法加载。

## 文件说明

| 文件 | 说明 |
|---|---|
| `.github/workflows/build.yml` | 构建工作流 |
| `device_defconfig` | 从设备导出的完整内核配置 |

## 协议

本项目使用 GPL-3.0 许可证。详见 [LICENSE](LICENSE)。

## Quest 3 固件 52433670036000520 / 内核 5.10.246（实验性）

另有独立的 **BakaSU LKM (Quest 3 5.10.246, experimental)** GitHub Actions 工作流：
[quest3-5.10.246.yml](.github/workflows/quest3-5.10.246.yml)。
原 **ReSukiSU LKM Build (Oculus Quest 3)** 工作流不变，继续对应旧版 5.10.240。

### 运行之前先导出当前设备的完整配置

在可运行 adb 的 **Linux / WSL** 中（避免 Windows PowerShell 5.1 重定向编码问题）：

```bash
adb exec-out 'zcat /proc/config.gz' > device_defconfig_5.10.246
grep -E 'CONFIG_(MODULES|MODVERSIONS|MODULE_SIG|FTRACE_SYSCALLS)' device_defconfig_5.10.246
```

将新生成的 `device_defconfig_5.10.246` 提交到运行工作流的同一分支，然后在 **Actions → BakaSU LKM (Quest 3 5.10.246, experimental) → Run workflow** 中选择该分支运行。若缺少此文件，工作流会在下载大体积源码和工具链之前失败。工作流也允许手动选择仓库原有的 `device_defconfig`，但那是旧版 5.10.240 的设备配置，不建议作为当前设备的正式构建依据。

### 重要的兼容性限制

- 已找到 Meta 官方精确对应固件 `5243367.3600.520` 的 [内核源码提交 `dbc2bc83a2b673f5d1c24ee9dadb0cd708ffda3f`](https://github.com/facebookincubator/oculus-linux-kernel/commit/dbc2bc83a2b673f5d1c24ee9dadb0cd708ffda3f)，其 Makefile 版本为 Linux 5.10.246。设备 `uname -r` 为 `5.10.246-gd7102a837402`；设备运行版本后缀与 Git 提交 SHA 不必相同。仍需验证运行内核与编译出的模块在符号 CRC 和 ABI 上兼容。
- 目标启用了 `CONFIG_MODVERSIONS=y`；即使 `kernelsu.ko` 的 `vermagic` 完全一致，也**不能保证模块符号 CRC、导出符号和实际内核 ABI 一致**。
- 新工作流使用现行 [BakaSU](https://github.com/Baka-SU/BakaSU) 源码，并固定可追溯的提交；不再依赖旧的 `ReSukiSU/ReSukiSU` 地址，也不再通过 `curl | bash` 执行远程脚本。
- 5.10.246 实验工作流现沿用此前成功构建时的 **ThinLTO + `vmlinux modules_prepare` + `KBUILD_MODPOST_WARN=1`** 逻辑。构建时从设备导出的 Full LTO 配置切换为 `CONFIG_LTO_CLANG_THIN=y`；不再强制生成 `Module.symvers`。这会降低构建时长，但 `modpost` 可能跳过未解析符号及 CRC 的检查。**CI 成功只能证明模块编译完成，不代表已验证模块在目标固件上可以安全加载。** 产物保留 `UNVERIFIED` 标识及 `BUILD-INFO.txt`，不会连接或修改设备。
- 如果之后取得准确的 Meta 内核源码 commit，可在手动运行时使用 `kernel_ref` 输入替换默认值。`bakasu_ref` 同样可以指定其他已验证版本。

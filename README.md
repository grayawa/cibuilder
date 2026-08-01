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

# AGENTS.md

本仓库为 **Oculus Quest 3** 构建 **ReSukiSU** `kernelsu.ko` 的 GitHub Actions 工作流。所有设备相关的取值都硬编码在 `.github/workflows/build.yml` 中，`device_defconfig` 是当前设备的内核配置。改动前先读懂下面两条规则。

## 核心规则

1. **vermagic 必须与设备完全一致**，否则 `kernelsu.ko` 无法加载。vermagic 由内核配置 `CONFIG_LOCALVERSION` + `CONFIG_LOCALVERSION_AUTO` 决定，必须等于设备 `uname -r`。
2. **必须对着设备的真实内核源码 + 设备导出的 `.config` 编译**。Quest 3 是 Meta 定制内核，Meta 破坏了 GKI 兼容性，通用 GKI/DDK 构建的模块无法加载。

## 换到其他设备/系统时要改的（按重要性排序）

### 1. `device_defconfig`（必须换）
从目标设备导出当前内核配置：

```sh
adb shell "zcat /proc/config.gz" > device_defconfig
```

### 2. workflow：内核源码分支/commit（必须换）
`build.yml` 中 `oculus-quest3-kernel-master` 分支和 `8f32bdea...` commit 是 Quest 3 专属。换成目标设备内核源码的仓库、分支和 commit。

### 3. workflow：`CONFIG_LOCALVERSION`（必须换）
`build.yml` 中的 `CONFIG_LOCALVERSION="-g9cb2f620a9a8"` 必须等于目标设备 `uname -r` 的后缀。不改则 vermagic 不匹配，模块加载失败。

### 4. workflow：禁用的驱动（可能需换）
当前禁用了 `QCA_CLD_WLAN`、`OCULUS`、`META`、`QCOM_KGSL`（Quest 3 上无法独立编译的 vendor 驱动）。换设备后按目标内核实际报错调整这些 `sed` 行——凡是编译失败、又对产物无影响的驱动都应禁掉。

### 5. workflow：clang 版本（可能需换）
当前用 Android clang 14（`llvm-r450784` 分支的 `clang-r450784c`）。应尽量与设备内核实际使用的编译器一致；不一致通常也能编出模块，但以能过编译为准。

### 6. 其他
- `device_defconfig` 里 `CONFIG_KSU` 若不存在会被追加为 `=m`，`CONFIG_KASAN` 会被强制关闭，这些一般无需改动。
- 若使用其他管理器，设备端包名 `com.resukisu.resukisu`（README 中的加载命令）要相应更改。

## 结构

```
.github/workflows/build.yml   # 构建工作流（改这里换设备）
device_defconfig              # 当前设备导出的内核配置
README.md                     # 面向使用者的说明
```

## 验证

产物 `kernelsu.ko` 的 vermagic 必须匹配设备。在构建产物上用 `modinfo`/`strings` 检查：

```sh
strings kernelsu.ko | grep vermagic
```

目标值示例（Quest 3）：`5.10.240-g9cb2f620a9a8 SMP preempt mod_unload modversions aarch64`

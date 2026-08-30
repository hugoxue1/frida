# Frida-Gum 源码定制日志

**定制目的**：消除 Frida-Gum 库中所有可被加固壳内存扫描检测到的特征字符串，使嵌入式注入不被梆梆加固（茅台）和 360 加固检测到。

**源码路径**：`E:/ckb_workspace/frida/subprojects/frida-gum/`
**Frida 版本**：17.8.0
**编译环境**：WSL Ubuntu-22.04，NDK r29-beta1，meson + ninja

## 一、定制内容：特征字符串替换（24 处，14 个文件）

### 1.1 运行时脚本路径（5 处）

将 QuickJS/V8 运行时脚本的路径前缀从 `/frida/` 改为 `/ckb/`，避免加固壳通过内存中的路径字符串检测到 Frida 运行时。

| 文件 | 替换 | 说明 |
|------|------|------|
| `bindings/gumjs/gumcmodule.c` | `/frida/` → `/ckb/`（5 处） | C 模块脚本路径 |

### 1.2 QuickJS 运行时相关（4 处）

| 文件 | 替换 | 说明 |
|------|------|------|
| `bindings/gumjs/gumquickcompile.c` | `frida/runtime` → `ckb/runtime` | 脚本编译路径 |
| `bindings/gumjs/gumquickscript.c` | `/_frida worker_runtime.js` → `/_ckb worker_runtime.js` | Worker 脚本路径 |
| `bindings/gumjs/gumquickscriptbackend.c` | `/frida/runtime/` → `/ckb/runtime/` + QuickJS 错误消息去 QuickJS 关键词（3 处） | 脚本后端路径 + 错误消息 |
| `bindings/gumjs/gumquickcore.c` | `"Frida"` → `"CKB"` | JS 全局对象名（QuickJS） |

### 1.3 V8 运行时相关（3 处）

| 文件 | 替换 | 说明 |
|------|------|------|
| `bindings/gumjs/gumv8script.cpp` | `/frida/runtime/` → `/ckb/runtime/` | V8 脚本路径 |
| `bindings/gumjs/gumv8scriptbackend.cpp` | `FRIDA_V8_EXTRA_FLAGS` → `CKB_V8_EXTRA_FLAGS` | V8 编译标志 |
| `bindings/gumjs/gumv8core.cpp` | `"Frida"` → `"CKB"` | JS 全局对象名（V8） |

### 1.4 Inspector Server（3 处）

| 文件 | 替换 | 说明 |
|------|------|------|
| `bindings/gumjs/guminspectorserver.c` | `Frida Agent` → `CKB Agent` 等（3 处） | Inspector server 标识字符串 |

### 1.5 ARM64 内存段标记（1 处）

| 文件 | 替换 | 说明 |
|------|------|------|
| `gum/backend-arm64/guminterceptor-arm64.c` | `__FRIDA_DATA` → `__CKB_DATA`（1 处） | ARM64 interceptor 数据段标记 |

### 1.6 Darwin only（4 处，不影响 Android 但保持一致性）

| 文件 | 替换 | 说明 |
|------|------|------|
| `gum/gumdarwingrafter.c` | `__FRIDA_` → `__CKB_`（3 处） | Darwin grafter 标记 |
| `gum/backend-darwin/gumcodesegment-darwin.c` | `frida-XXXXXX.dylib` → `ckb-XXXXXX.dylib` | Darwin 代码段临时文件名 |
| `gum/backend-darwin/gumdarwinmapper.c` | `frida_dylib_range=` → `ckb_dylib_range=` | Darwin 映射器参数 |

### 1.7 编译环境兼容（1 处）

| 文件 | 替换 | 说明 |
|------|------|------|
| `releng/env_android.py` | NDK_REQUIRED 从 25 改为 29 | 兼容 WSL 的 r29 NDK |

## 二、编译流程

### 2.1 WSL 环境配置

```bash
export PATH=$HOME/.local/bin:/usr/local/bin:/usr/bin:/bin
export ANDROID_NDK_ROOT=$HOME/Android/Sdk/ndk/android-ndk-r29-beta1
cd /mnt/e/ckb_workspace/frida/subprojects/frida-gum
rm -rf build
bash configure --host android-arm64 --with-frida-version 17.8.0 --enable-gumjs
```

### 2.2 编译

```bash
cd /mnt/e/ckb_workspace/frida/subprojects/frida-gum/build
ninja -j4
```

### 2.3 生成产物

```
build/gum/libfrida-gum-1.0.a              # 核心库（7.4MB）
build/bindings/gumjs/libfrida-gumjs-1.0.a  # gumjs 绑定（17MB）
build/libs/gum/prof/libfrida-gum-prof-1.0.a
build/libs/gum/heap/libfrida-gum-heap-1.0.a
```

### 2.4 合并为完整 .a + 符号前缀

```bash
# 合并 frida .a + 第三方 .a
bash /mnt/e/ckb_workspace/frida_gum_build/merge_devkit.sh

# 给第三方库符号加 _frida_ 前缀
bash /mnt/e/ckb_workspace/frida_gum_build/prefix_symbols.sh
```

## 三、.so 二进制补丁（编译后）

编译后的 libgumjs.so 仍包含 Frida 库 .rodata 段中的 GObject 类型名（`GumInterceptor`、`GumStalker` 等）和 .dynstr 段中的 C++ mangled name。用 `patch_so.py` 做二进制补丁：

```bash
python3 /mnt/e/ckb_workspace/frida_gum_build/patch_so.py
```

补丁替换规则（同长度，安全）：
- `Gum` → `Ckb`（.rodata + .dynstr）
- `gum_` → `ckb_`（.rodata）
- `GUM_` → `CKB_`（.rodata）
- `QuickJS` → `QckCkbS`（.rodata）

保护规则（跨模块符号不替换）：
- `onCkbEngineRegister` — dlsym 查找/导出符号名
- `CkbEngine` — C++ 类名
- `ckb_hooks` — 脚本目录路径

## 四、验证

补丁后全文件搜索 `Gum`（排除 Friday）= 0，`QuickJS` = 0，`gum_` = 0。

茅台 app（梆梆加固）和 360 加固 app 均稳定运行不崩溃。

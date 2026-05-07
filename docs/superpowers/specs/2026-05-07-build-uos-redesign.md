# 设计文档：Build UOS20 兼容 AppImage（重构版）

**日期**: 2026-05-07
**项目**: Ghostty 终端模拟器
**目标**: 在 GitHub Actions 上构建可在 UOS20（基于 Debian 10、glibc 2.28、内网无法装包）上独立运行的 Ghostty
**取代**: `2026-04-29-build-uos-design.md`（旧方案试图在 debian:10 里源码编译整个 GTK4 stack，已证明不可行）

---

## 1. 背景：为什么换方案

旧方案在 `debian:10` 容器里从源码编译 zlib → libpng → libffi → glib → wayland → wayland-protocols → libxkbcommon → pixman → cairo → pango → gdk-pixbuf → libepoxy → graphene → GTK4 → gtk4-layer-shell，再编译 ghostty。

实测中已经在 pango 这一步因为 fontconfig 子项目要 xft/cairo-ft 失败；而 debian:10 的 `glib 2.58` 比 GTK4 要求的 `>= 2.66` 还低，意味着 glib 也得自己重编。链路上还有 GTK4 自带很多子项目、libadwaita 1.4+ 没在任何 buster 仓库，最终路径会越走越长且每一步都易碎。

这条路本质上是在重新发明 Linux 发行版。

## 2. 关键洞察

1. **Zig 自带 glibc 多版本头文件**。`-Dtarget=x86_64-linux-gnu.2.28` 会让 Zig 链接器把 ghostty 主二进制对 glibc 符号的引用全部限制到 `GLIBC_2.28` 及以下版本。**构建机的实际 glibc 版本无所谓**。
2. **Ghostty 大部分 C 依赖已经是 Zig 包**：fontconfig、freetype、harfbuzz、libpng、zlib、libintl、oniguruma、wuffs、glslang、spirv-cross 都通过 `build.zig.zon` 声明，由 Zig build 自己编译并静态链接到 ghostty 中，glibc 也按 target 控制。
3. **真正需要系统提供的只有 GTK4 stack 的动态库**：libgtk-4、libadwaita-1、libgtk4-layer-shell-0、libglib/gio/gobject、libpango、libcairo、libgdk_pixbuf、libepoxy、libgraphene、libxkbcommon、libwayland-client/cursor、以及它们传递依赖的 .so（libfontconfig、libfreetype、libharfbuzz 在 GTK 端的副本、libpixman、libfribidi 等）。
4. **AppImage / 自包含目录的标准做法**就是把这些 .so 全部 bundle 到产物里，运行时通过 RPATH `$ORIGIN/lib` 让动态链接器在自己目录找。这是 Inkscape、Krita、OBS 等众多软件多年验证的方案。
5. **构建环境必须保证 GTK 库本身不引入 glibc > 2.28 的符号**。Debian 10 的 GTK4 不存在；Debian 11 的 GTK4 (4.6) 链接到 glibc 2.31 的 .so；Debian 12 GTK4 (4.8) 链接到 glibc 2.36；Debian 13 GTK4 (4.14) 链接到 glibc 2.38。这些 .so 的 glibc 符号需求会成为 UOS20 上运行的硬限制。

## 3. 选定方案

**两阶段构建**：

**阶段 1：在 `debian:11-slim` 构建主体**
- glibc 2.31，符合"几乎是 2.28"（差异极小，绝大多数 GTK4 .so 实际只用到 2.28 已有的符号；少量引入 2.31 符号的库可以打补丁/换源/降级）。
- 系统包 `libgtk-4-dev` (4.6.9), `libadwaita-1-dev` (1.0.x), `libgtk4-layer-shell-dev`, `libwayland-dev`, `libxkbcommon-dev`, `blueprint-compiler`（如有），编译期足够用。
- Zig 0.15.2 用 `-Dtarget=x86_64-linux-gnu.2.28 -Dcpu=x86_64_v2` 编译 ghostty，强制主二进制只依赖 glibc 2.28。

**阶段 2：打包成自包含目录**
- 用 `ldd` 递归收集所有非系统级库（即：除 `linux-vdso`, `ld-linux-*`, `libc`, `libm`, `libdl`, `libpthread`, `libresolv`, `librt` 这些 glibc 自带的之外的全部 .so）。
- 复制到 `lib/` 子目录，`patchelf --set-rpath '$ORIGIN/lib'` 把主二进制和所有 bundle 的 .so 的 RPATH 都改写。
- 用 `objdump -T` 校验所有 bundle 的 .so 引用的最高 glibc 符号版本，列出超过 2.28 的库（如有）让我们决定怎么处理。
- 输出 `ghostty-uos-amd64.tar.gz`（直接解压双击运行）。后续如有需要再加 AppImage 封装，初版 tarball 即可满足"双击运行"。

**libadwaita 版本风险**：debian:11 的 libadwaita 是 1.0.x，ghostty 运行时通过 `getRuntimeVersion()` 检查能力降级（代码已有 `versionAtLeast` 机制），但**编译期** `@cInclude("adwaita.h")` 要求头文件存在新符号。如果 ghostty 代码直接调用了 1.4+ 才有的 API，编译会失败。需要测试，如失败则升级到 `debian:12-slim`（glibc 2.36）并接受少量 .so 引入 2.30+ 符号（依赖 patchelf 替换符号或退而求其次构建一个 glibc 2.31 兜底拷贝；但优先让 2.28 路线走通看看 ghostty 实际报错）。

## 4. 失败时的回退梯度

按优先级尝试：

1. **debian:11 + GTK4 1.0** → 如果 ghostty 用了 adw 1.2+ API 编译失败
2. **debian:12 + GTK4 4.8 + adw 1.2** → 部分 .so 引入 glibc 2.30/2.32 符号；用 `zig cc` 重新链接缺的少量库，或检查是否真用到这些符号
3. **debian:12 + 自编译 libadwaita 1.4** → 系统 GTK4 + 自编 adw（adw 依赖只有 GTK4/glib，自编一个比 pango 好编多了）
4. **预编译 GTK4 sysroot**（最差兜底，留作备案不在初版做）

## 5. yml 文件结构

```yaml
name: Build for UOS20 (glibc 2.28)
on:
  push:
    branches: ['**']
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    container:
      image: debian:11-slim
      options: --user root
    env:
      DEBIAN_FRONTEND: noninteractive
    steps:
      - checkout
      - apt install: build-essential, pkg-config, patchelf, file, binutils,
                     curl, ca-certificates, git,
                     libgtk-4-dev, libadwaita-1-dev, libgtk4-layer-shell-dev,
                     libwayland-dev, libxkbcommon-dev,
                     blueprint-compiler  # 可能需要从 backports
      - setup-zig 0.15.2
      - zig build -Doptimize=ReleaseSafe \
                  -Dtarget=x86_64-linux-gnu.2.28 \
                  -Dcpu=x86_64_v2 \
                  -Dapp-runtime=gtk \
                  -Dgtk-x11=false -Dgtk-wayland=true \
                  -Dpie=true
      - 收集运行时依赖（递归 ldd），过滤 glibc 库
      - 复制到 dist/ghostty-uos-amd64/lib/
      - patchelf --set-rpath '$ORIGIN/lib' 主二进制和所有 lib/*.so
      - 用 objdump -T 验证 glibc 符号版本，打印报告
      - tar 打包
      - upload-artifact
```

## 6. 验证步骤

每次构建产物完成后，workflow 自动跑以下检查（fail-fast）：

1. `ldd dist/ghostty-uos-amd64/ghostty` 不能有 "not found"
2. `ldd dist/ghostty-uos-amd64/ghostty | grep -v 'linux-vdso\|ld-linux\|/lib/x86_64-linux-gnu/libc\|libm\|libdl\|libpthread\|libresolv\|librt'` 全部应当指向 `dist/ghostty-uos-amd64/lib/`
3. `objdump -T dist/ghostty-uos-amd64/ghostty | grep GLIBC_ | awk '{print $5}' | sort -V | uniq` 输出最高版本必须 ≤ `GLIBC_2.28`
4. 对 `dist/ghostty-uos-amd64/lib/*.so` 同样跑 #3，统计超过 2.28 的库到 summary，如全部满足则视为完整通过；否则把超标库列出来作为后续工作

## 7. 成功标准

- workflow run 全绿
- artifact `ghostty-uos-amd64.tar.gz` 可下载
- 验证步骤 1, 2, 3 全通过
- 验证步骤 4 报告中超标库数量为 0，或者列表足够小可以人工逐个评估
- (可选事后验证) 用户在 UOS20 机器上解压运行 `./ghostty --version` 不报 GLIBC_X.YY not found

## 8. 不做的事（YAGNI）

- 不做 AppImage runtime 封装（tarball 已足够"双击运行"）
- 不构建 arm64
- 不打 .deb / .desktop（用户没要求）
- 不做发布到 Release（仅 artifact）
- 不缓存 apt 包（apt install 在 debian:11 很快，缓存反而增加复杂度）

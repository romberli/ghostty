# 设计文档：GitHub Action 构建 UOS20 兼容版本

**日期**: 2026-04-29
**项目**: Ghostty 终端模拟器
**目标**: 创建可在统信UOS20 (基于Debian 10) 上独立运行的版本

---

## 1. 目标与约束

### 用户需求
- 在统信UOS20 amd64系统上运行Ghostty
- UOS20是基于Debian 10二次开发的系统
- 本地无对应构建环境，需使用GitHub Action
- 目标系统在内网环境，无法安装任何软件包
- 系统仅保证存在glibc 2.38

### 构建目标
- 完整的Ghostty应用程序（包含GTK GUI）
- 不依赖操作系统的任何软件包（除glibc外）
- 双击可运行，无需启动脚本或额外配置

---

## 2. 整体架构

### 工作流结构
```
Push触发 → 检查缓存 → [无缓存]构建依赖 → 编译Ghostty → 打包产物 → 发布Artifacts
            ↓           ↓
          [有缓存]直接使用 → 编译Ghostty → 打包产物 → 发布Artifacts
```

### 核心策略
- 使用 `debian:10` 容器作为构建环境，确保最大兼容性
- 分两层缓存机制避免重复构建
- 使用 ELF RPATH 技术让程序自动定位库文件
- 最终产物完全自包含

### 时间预估
- 首次构建：约1-2小时（编译GTK4等所有依赖）
- 后续构建：约15-20分钟（仅编译Ghostty代码）

---

## 3. 依赖构建流程

### 构建工具
从源码或预编译包安装：
- **Zig 0.13.x**: 官方预编译二进制，安装到 `/opt/zig`
- **Meson**: Python pip安装
- **Ninja**: 从源码编译

### 依赖编译顺序（全部从源码）

**基础层**:
- zlib
- libffi
- libintl (gettext)

**核心层**:
- glib (静态)
- libxml2
- libxkbcommon

**图形层**:
- freetype (Ghostty已自带)
- harfbuzz (Ghostty已自带)
- libpng (Ghostty已自带)
- cairo (静态)
- pango (静态)

**GTK层**:
- libepoxy
- graphene
- gtk4 (静态)
- gtk4-layer-shell

**Wayland层**:
- wayland
- wayland-protocols

### 构建策略
1. 每个依赖配置为静态库优先：`--default-library=static`
2. 所有依赖编译到统一目录 `/opt/deps`
3. 使用 `-Dprefix=/opt/deps` 统一安装路径
4. 编译时使用 `-O2` 优化
5. GTK4使用静态链接模式

---

## 4. 产物打包与运行

### RPATH技术实现

**编译时设置RPATH**:
```
-Wl,-rpath,'$ORIGIN/lib'
```

`$ORIGIN` 是ELF特殊变量，动态链接器会自动解析为可执行文件所在目录。

### 最终产物结构

```
ghostty-uos-amd64/
├── ghostty          (主程序，RPATH=$ORIGIN/lib)
└── lib/             (必需动态库)
    ├── libgtk-4.so
    ├── libglib-2.0.so
    ├── libcairo.so
    ├── libpango-1.0.so
    ├── libpangocairo-1.0.so
    ├── libgdk_pixbuf-2.0.so
    ├── libgio-2.0.so
    ├── libgmodule-2.0.so
    ├── libgobject-2.0.so
    ├── libepoxy.so
    ├── libgraphene-1.0.so
    ├── libxkbcommon.so
    ├── libwayland-client.so
    ├── libwayland-cursor.so
    ├── libffi.so
    ├── libz.so
    └── ... (其他必需动态库)
```

### 使用方式
- 解压到任意目录
- 直接双击 `ghostty` 即可运行
- 程序自动在同级 `lib/` 目录查找动态库
- 仅依赖系统glibc

### 验证机制
- 构建完成后用 `ldd ghostty` 检查依赖
- 确保所有库指向 `$ORIGIN/lib` 或系统glibc
- 使用 `patchelf` 确认RPATH设置正确

---

## 5. GitHub Action Job结构

### 工作流定义 (build-uos.yml)

```yaml
触发条件:
  - push: 所有分支
  - workflow_dispatch: 手动触发（备用）

Jobs:
  build-uos:
    runs-on: ubuntu-latest
    container: debian:10
    
    steps:
      1. 检出代码 (actions/checkout@v4)
      2. 检查缓存（Zig + GTK依赖）
      3. [无缓存] 安装基础工具（apt, pip）
      4. [无缓存] 下载并安装Zig编译器
      5. [无缓存] 从源码编译所有依赖（按依赖顺序）
      6. [无缓存] 保存缓存到GitHub Actions Cache
      7. 编译Ghostty (zig build)
      8. 收集依赖库到lib/目录（使用ldd检测）
      9. 设置RPATH（patchelf -Wl,-rpath,'$ORIGIN/lib'）
      10. 验证依赖（ldd检查）
      11. 打包为tar.gz
      12. 发布Artifacts（下载链接有效期90天）
```

### 缓存策略
- **Key**: `uos-deps-v1-${{ hashFiles('build.zig.zon.json') }}`
  - 依赖版本变化才重建缓存
- **Path**: 
  - `/opt/zig` (Zig编译器)
  - `/opt/deps` (所有编译的依赖库)
- **恢复Key**: `uos-deps-v1-` (尝试使用旧版本缓存)
- 缓存有效期：7天（GitHub默认）

### 构建优化
- 并行编译：meson配置 `-Doptimization=2`
- ninja并行构建：`ninja -j$(nproc)`
- 合理利用GitHub Actions并发限制（默认2核）

---

## 6. 实现细节

### Docker容器配置
```yaml
container:
  image: debian:10
  options: --user root
  env:
    DEBIAN_FRONTEND: noninteractive
    TZ: UTC
```

### 基础工具安装（Debian 10容器内）
```bash
apt-get update
apt-get install -y \
  build-essential \
  python3 python3-pip \
  git \
  curl wget \
  pkg-config \
  autoconf automake libtool \
  patchelf
pip3 install meson
```

### Zig安装
```bash
ZIG_VERSION=0.13.0
curl -L https://ziglang.org/download/${ZIG_VERSION}/zig-linux-x86_64-${ZIG_VERSION}.tar.xz \
  | tar -xJ -C /opt
ln -s /opt/zig-linux-x86_64-${ZIG_VERSION}/zig /usr/local/bin/zig
```

### 依赖编译示例（glib）
```bash
# 安装到统一目录
export PREFIX=/opt/deps
export PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig
export LD_LIBRARY_PATH=$PREFIX/lib

# 编译libffi
wget https://github.com/libffi/libffi/releases/download/v3.4.6/libffi-3.4.6.tar.gz
tar xf libffi-3.4.6.tar.gz
cd libffi-3.4.6
./configure --prefix=$PREFIX --enable-static --disable-shared
make -j$(nproc) && make install

# 编译glib
wget https://download.gnome.org/sources/glib/2.78/glib-2.78.4.tar.xz
tar xf glib-2.78.4.tar.xz
cd glib-2.78.4
meson setup build \
  --prefix=$PREFIX \
  --default-library=static \
  -Dlibffi=enabled
ninja -C build && ninja -C build install
```

### Ghostty编译
```bash
export PKG_CONFIG_PATH=/opt/deps/lib/pkgconfig
export LD_LIBRARY_PATH=/opt/deps/lib
zig build \
  -Dtarget=x86_64-linux-gnu.2.38 \
  --search-prefix /opt/deps \
  --search-prefix /usr
```

### 收集依赖库
```bash
# 创建lib目录
mkdir -p lib

# 获取ghostty所需的动态库列表
ldd zig-out/bin/ghostty | grep "=> /opt/deps" | awk '{print $3}' | while read lib; do
  cp "$lib" lib/
done

# 复制库的符号链接目标
for lib in lib/*.so; do
  if [ -L "$lib" ]; then
    cp $(readlink -f "$lib") lib/
  fi
done
```

### 设置RPATH
```bash
patchelf --set-rpath '$ORIGIN/lib' zig-out/bin/ghostty
```

### 最终打包
```bash
mkdir ghostty-uos-amd64
mv zig-out/bin/ghostty ghostty-uos-amd64/
mv lib ghostty-uos-amd64/
tar czf ghostty-uos-amd64.tar.gz ghostty-uos-amd64
```

---

## 7. 风险与缓解

### 潜在问题

1. **构建时间过长**
   - 首次构建可能接近GitHub Actions 6小时限制
   - 缓解：合理拆分步骤，使用缓存

2. **glibc版本不匹配**
   - Debian 10默认glibc 2.28，UOS20用户确认有2.38
   - 缓解：编译时指定glibc版本符号版本

3. **GTK4静态链接不完全**
   - GTK4某些模块可能强制动态加载
   - 缓解：将这些模块的.so文件打包到lib/

4. **依赖遗漏**
   - ldd可能漏掉某些运行时加载的库
   - 缓解：构建后测试运行，补充缺失库

---

## 8. 成功标准

构建成功的判断标准：
1. GitHub Action完成无错误
2. 产物ghostty-uos-amd64.tar.gz可下载
3. 解压后ldd显示所有依赖指向$ORIGIN/lib或系统glibc
4. 在Debian 10容器内可运行ghostty --version
5. 程序可显示GTK窗口（需X11/Wayland环境）

---

## 9. 后续改进

可选的后续优化：
1. 添加workflow_dispatch手动触发参数（选择优化级别）
2. 使用GitHub Release发布稳定版本
3. 添加更多测试步骤（虚拟 framebuffer运行）
4. 创建.desktop文件方便桌面集成
5. 构建多架构支持（arm64等）
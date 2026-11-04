# Linux 系统中的 AppImage 包

- 日期：2026-09-02
- 标签：linux, appimage, 概念
- 关联笔记：[Linux Electron 应用](2026-09-03-linux-Electron应用.md)、[Windows 目录结构与快捷指令](2026-09-12-windows-目录结构与快捷指令.md)

## 什么是 AppImage？

**AppImage** 是一种 **便携式软件包格式**，它将应用程序及其所有依赖打包到一个独立的可执行文件中，用户无需安装即可在 Linux 系统上运行。

## AppImage 的特点

| 特性         | 说明                                           |
| ---------- | -------------------------------------------- |
| 🚀 即下即用    | 下载后赋予执行权限即可运行，无需安装                           |
| 📦 自包含     | 包含应用所需的所有依赖库                                 |
| 🐧 跨发行版    | 支持大多数 Linux 发行版（Ubuntu、Fedora、Debian、Arch 等） |
| 🔒 无需 root | 普通用户即可运行                                     |
| 💾 不修改系统   | 不会污染系统环境，删除即卸载                               |

## AppImage 文件格式

- 文件后缀：`.AppImage`
- 本质上是一个 **压缩的镜像文件**（squashfs 格式）
- 下载后是一个可执行文件

## 如何使用 AppImage

### 基本使用步骤

```bash
# 1. 下载 AppImage 文件（例如从 GitHub Releases）
wget https://example.com/application.AppImage

# 2. 赋予可执行权限
chmod +x application.AppImage

# 3. 直接运行
./application.AppImage
```

### 双击运行

在文件管理器中右键 AppImage 文件 → 属性 → 权限 → 勾选"允许作为程序执行" → 双击即可

## AppImage 与其他包格式对比

| 包格式 | 安装方式 | 适用范围 |
|--------|---------|---------|
| `.deb` | `dpkg -i` 或 `apt install` | Debian/Ubuntu 系列 |
| `.rpm` | `rpm -ivh` 或 `yum install` | RedHat/Fedora/CentOS |
| **`.AppImage`** | **无需安装，直接运行** | **所有 Linux 发行版** |
| `.Snap` | 通过 snapd 安装 | Ubuntu 及大部分发行版 |
| `.Flatpak` | 通过 flatpak 安装 | 大部分发行版 |

## 常用 AppImage 工具

以下应用官方提供 AppImage 下载：
- **图像处理**：Krita
- **视频剪辑**：OpenShot
- **实用工具**：Bitwarden、Balena Etcher（仅旧版 1.x 提供 AppImage，2.x 起官方已不再发布）

> 注意：VS Code、IntelliJ IDEA、Postman、Firefox 等热门应用**官方并不提供 AppImage**，网上流传的多为社区第三方打包，使用前需自行验证来源。

## AppImage 辅助工具

### 1. AppImageLauncher
自动管理 AppImage 文件（集成到应用菜单、可执行权限设置）

### 2. AppImageUpdate
独立的更新工具（并非 AppImage 内置参数），包含 Qt 图形界面 `AppImageUpdate` 和命令行工具 `appimageupdatetool`：
```bash
# 命令行更新某个 AppImage（前提：该 AppImage 打包时嵌入了更新信息，如 zsync 地址）
./appimageupdatetool-x86_64.AppImage ./application.AppImage
```
> 注意：electron-builder 打包的 AppImage 使用自己的更新机制，不能用上述工具更新。

### 3. 解压查看内容
`--appimage-extract` 是 AppImage 运行时内置参数，无需额外工具（独立的 AppImageExtract 工具仅用于老旧的 type 1 AppImage，已弃用）：
```bash
# 解压 AppImage 文件查看内容
./tool.AppImage --appimage-extract
```

## 在 AppImage 内执行命令

```bash
# 解压并进入目录
./application.AppImage --appimage-extract
cd squashfs-root
ls  # 查看应用文件结构
```

## 注意事项

⚠️ **可能遇到的问题**：

1. **FUSE 库未安装**（较新系统常见）
```bash
# Ubuntu/Debian（Ubuntu 24.04 起包名改为 libfuse2t64）
sudo apt install libfuse2

# 如果无法安装 FUSE，可临时解压后运行（退出后自动清理）：
./application.AppImage --appimage-extract-and-run
```

2. **磁盘空间占用**：AppImage 自包含全部依赖，文件本身体积较大。正常运行时通过 FUSE 只读挂载镜像，并不会整体解压；只有使用 `--appimage-extract-and-run` 兜底时才会临时解压到 /tmp

3. **桌面集成**：原生文件管理器没有"集成到系统"菜单，需安装 AppImageLauncher 后双击才会提示集成；也可使用 `appimaged` 守护进程自动管理（官方 appimaged 仓库已归档，目前推荐 AppImageLauncher）

## 创建 AppImage 的工具

如果你想把自己的应用打包成 AppImage，可以使用：
- **linuxdeployqt** - Qt 应用打包
- **AppImageKit** - 通用打包工具
- **electron-builder** - Electron 应用打包

## 总结

**AppImage** 是一种 "绿色版" 的 Linux 软件分发格式，最大的优势是 **零安装、跨发行版、可便携使用**，特别适合需要在多种 Linux 环境使用的应用。
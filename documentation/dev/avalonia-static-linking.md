# Avalonia 静态链接库构建与自动化工作流指南

本文档介绍针对 Avalonia UI 在 .NET NativeAOT 场景下使用的 **SkiaSharp** 和 **HarfBuzzSharp** 原生静态链接库（`.lib` / `.a`）的构建工作流、Avalonia 新版本自动监听发布机制，以及上游 `mono/SkiaSharp` 仓库的自动同步机制。

---

## 1. 架构与原理

### 1.1 为什么需要静态链接库
默认情况下，SkiaSharp 和 HarfBuzzSharp 通过 P/Invoke 动态加载外部动态链接库（Windows 下为 `libSkiaSharp.dll` / `libHarfBuzzSharp.dll`，Linux 下为 `.so`，macOS 下为 `.dylib`）。
在 .NET NativeAOT 编译为单一可执行文件（Single-File Executable）时，若要将图形渲染引擎完整内嵌到单一二进制文件中，需要：
1. 原生编译的静态库（`skia.lib` + `SkiaSharp.lib` + `libHarfBuzzSharp.lib` 或 Unix 下的 `.a`）。
2. 在 MSBuild 项目中声明 `<DirectPInvoke Include="libSkiaSharp" />` 和 `<DirectPInvoke Include="libHarfBuzzSharp" />`，指示 AOT 编译器在静态编译期直接解析原生符号。
3. 引入 `<NativeLibrary Include="..." />` 提供原生库文件路径。

### 1.2 自动化组件一览
本仓库包含以下三组核心工作流与配置：

| 组件 | 文件路径 | 用途 |
|---|---|---|
| **静态库构建工作流** | `.github/workflows/build-static-libs.yml` | 跨平台（Win/Linux/macOS）编译静态库、打包 NuGet 包并创建 GitHub Release |
| **Avalonia 更新监听** | `.github/workflows/auto-avalonia-sync.yml` | 定时检测 Avalonia 最新 Release，解析其 Skia 版本并自动触发静态库构建 |
| **上游仓库同步** | `.github/workflows/auto-upstream-sync.yml` | 每日定时从 `mono/SkiaSharp` 同步 `main` 分支代码与 Git Tags |
| **GN 构建配置** | `native/static/args.*.gn` | 针对各平台的 Skia 静态编译参数（开启 `is_static_skiasharp = true` 等） |
| **NuGet 打包工程** | `nuget/SkiaSharp.Static/` | 将各平台静态库打入统一 NuGet 包，并提供自动注入 DirectPInvoke 的 targets |

---

## 2. 工作流运作说明

### 2.1 静态库构建工作流 (`build-static-libs.yml`)
- **触发方式**：
  - **手动触发**（GitHub Actions 页面点击 `Run workflow`）：
    - `avalonia_version`：目标 Avalonia 版本（例如 `12.1.2`）
    - `skiasharp_version`：目标 SkiaSharp 版本（例如 `3.119.4`）
    - `skiasharp_ref`：编译时检出的分支或 Tag（例如 `release/3.119.4`）
    - `harfbuzz_version`：HarfBuzzSharp 版本（例如 `8.3.1.3`）
    - `package_id`：NuGet Package ID（默认 `Summpot.SkiaSharp.Static`）
    - `publish_release`：是否创建 GitHub Release 并挂载资产（默认 `true`）
    - `publish_nuget`：是否通过 NuGet.org **Trusted Publishing**（OIDC 免密认证）自动推送到 NuGet.org（默认 `true`）
    - `nuget_user`：NuGet.org 用户名（默认 `Summpot`）
    - `skip_native_builds`：是否跳过原生编译，直接复用已发布 Release 的产物进行打包/推送（默认 `false`）
  - **工作流调用**（由 `auto-avalonia-sync.yml` 作为 Reusable Workflow 自动调用）
- **产物与分发**：
  - 独立平台归档包：
    - `SkiaSharp.Static-win-x64.zip`
    - `SkiaSharp.Static-linux-x64.tar.gz`
    - `SkiaSharp.Static-osx.tar.gz`
  - NuGet 统一安装包：
    - `Summpot.SkiaSharp.Static.<version>.nupkg`
  - GitHub Release：自动创建 Tag `avalonia-<AvaloniaVersion>-skia-<SkiaVersion>` 并挂载所有产物。
  - NuGet.org 发布：利用 GitHub OIDC 与 `NuGet/login@v1` 交换短期临时 API 密钥，免密安全推送到官方 NuGet 源。

### 2.2 Avalonia 更新自动监听 (`auto-avalonia-sync.yml`)
- 每 6 小时自动运行一次（亦可手动立即触发）。
- 流程：
  1. 调用 GitHub API 查询 `AvaloniaUI/Avalonia` 最新 Release。
  2. 提取该版本引用的 `SkiaSharp` 和 `HarfBuzzSharp` 版本号。
  3. 检查当前仓库是否已发布过对应版本的 Release。
  4. 若检测到尚未构建的新版本，自动调用 `build-static-libs.yml` 进行全平台构建与自动发布！

### 2.3 上游仓库同步 (`auto-upstream-sync.yml`)
- 每天 02:00 UTC 自动运行（亦可手动立即触发）。
- 流程：
  1. 获取上游 `https://github.com/mono/SkiaSharp.git` 的最新提交与所有 Tags。
  2. 比较 `upstream/main` 与本仓库 `origin/main`。
  3. 若无本地分叉冲突，执行 Fast-Forward 快进合并并推送到 `origin/main`，同时同步 Tags。
  4. 若存在冲突，自动创建分支 `automation/sync-upstream-YYYYMMDD` 并提交 Pull Request，便于维护者审查。

---

## 3. 在 Avalonia 项目中集成静态链接库

使用发布的 NuGet 包非常简单，只需在 Avalonia 应用程序中引入即可，无需修改业务代码。

### 方式 A：通过 `Directory.Build.props` 全局配置（推荐）

在解决方案根目录创建或编辑 `Directory.Build.props`：

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <!-- 发布配置 -->
  <PropertyGroup Label="PublishConfiguration" Condition="'$(Configuration)' == 'Release'">
    <PublishAot>true</PublishAot>
    <TrimMode>full</TrimMode>
  </PropertyGroup>

  <!-- 引入静态库包 -->
  <ItemGroup Label="StaticSkiaSharp">
    <PackageReference Include="Summpot.SkiaSharp.Static" Version="3.119.4-avalonia.12.1.2" />
  </ItemGroup>
</Project>
```

> **提示**：引入 `Summpot.SkiaSharp.Static` 后，该包内置的 `targets` 会在 `PublishAot == true` 时自动注入 `<DirectPInvoke>` 以及对应平台的静态库和底层系统依赖（如 Windows 的 `d3d12.lib`、`dxgi.lib` 等，Linux 的 `-lfontconfig` 等，macOS 的 Metal 框架）。

### 方式 B：在 `.csproj` 中直接引用

```xml
<ItemGroup>
  <PackageReference Include="Summpot.SkiaSharp.Static" Version="3.119.4-avalonia.12.1.2" />
</ItemGroup>
```

### 发布单一可执行文件

使用标准 .NET NativeAOT 命令发布：

```bash
# Windows x64
dotnet publish -r win-x64 -c Release /p:PublishAot=true

# Linux x64
dotnet publish -r linux-x64 -c Release /p:PublishAot=true

# macOS arm64
dotnet publish -r osx-arm64 -c Release /p:PublishAot=true
```

发布完成后，输出目录中的单一执行程序（如 `MyApp.exe`）即已完全内嵌 SkiaSharp 和 HarfBuzzSharp，不再依赖任何外部 `libSkiaSharp` 动态库。

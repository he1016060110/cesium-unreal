# OnsiteTemplate 项目 - Windows 开发环境设置

针对 OnsiteTemplate 项目的 Cesium for Unreal 开发环境设置指南（Windows）。

## 项目路径说明

```
项目根目录: D:\data\onsite\OnsiteTemplate
Cesium 插件: D:\data\onsite\OnsiteTemplate\Plugins\cesium-unreal
cesium-native: D:\data\onsite\OnsiteTemplate\Plugins\cesium-unreal\extern\cesium-native
Unreal Engine: 5.5
```

## 前置条件

- 安装 CMake (3.15 或更高版本): https://cmake.org/install/
- 安装 Visual Studio 2022 v17.4+
  - **Workloads** 勾选 `Desktop development with C++`
  - **Workloads** 勾选 `Game development with C++`
  - **Individual components** 勾选 `.NET Framework 4.8 SDK`
  - **Individual components** 勾选 `MSVC v143 - VS2022 C++ x64/x86 build tools (v14.38.33130)` ⚠️ 关键
- 安装 .NET Core 3.1 Runtime: https://dotnet.microsoft.com/en-us/download/dotnet/3.1
- (可选) 安装 [nasm](https://www.nasm.us/) 以获得更好的 JPEG 解码性能
- 安装 Unreal Engine 5.5

## 安装指定版本的 MSVC 工具链

> ⚠️ **重要**: 必须安装 v14.38.33130 版本的 MSVC 工具链，否则编译会失败。

1. 打开 **设置 → 应用 → 已安装的应用**
2. 找到 **Visual Studio Professional/Enterprise/Community 2022**，点击 **修改**
3. 切换到 **单个组件 (Individual components)** 标签页
4. 在搜索框中输入 `14.38`
5. 勾选 **MSVC v143 - VS2022 C++ x64/x86 build tools (v14.38.33130)**
6. 点击 **修改** 完成安装

## 配置 Unreal Build Tool 使用指定工具链

创建或编辑文件 `%AppData%\Unreal Engine\UnrealBuildTool\BuildConfiguration.xml`：

```xml
<?xml version="1.0" encoding="utf-8" ?>
<Configuration xmlns="https://www.unrealengine.com/BuildConfiguration">
    <WindowsPlatform>
        <CompilerVersion>14.38.33130</CompilerVersion>
    </WindowsPlatform>
</Configuration>
```

> **提示**: 在 PowerShell 中，`%AppData%` 对应 `$env:AppData`，通常是 `C:\Users\你的用户名\AppData\Roaming`

可以使用以下 PowerShell 命令快速创建：

```powershell
$configDir = "$env:AppData\Unreal Engine\UnrealBuildTool"
$configFile = "$configDir\BuildConfiguration.xml"

# 创建目录（如果不存在）
if (!(Test-Path $configDir)) {
    New-Item -ItemType Directory -Path $configDir -Force
}

# 写入配置文件
@'
<?xml version="1.0" encoding="utf-8" ?>
<Configuration xmlns="https://www.unrealengine.com/BuildConfiguration">
    <WindowsPlatform>
        <CompilerVersion>14.38.33130</CompilerVersion>
    </WindowsPlatform>
</Configuration>
'@ | Out-File -FilePath $configFile -Encoding utf8

Write-Host "配置文件已创建: $configFile"
```

## 更新 cesium-native 子模块

如果 `extern\cesium-native` 目录为空或缺失，运行：

```powershell
cd D:\data\onsite\OnsiteTemplate\Plugins\cesium-unreal
git submodule update --init --recursive
```

## 构建 cesium-native

### 方法一：使用 x64 Native Tools Command Prompt（推荐）

1. 从开始菜单打开 **x64 Native Tools Command Prompt for VS 2022**

2. 设置环境变量指定工具链版本和 UE 路径：

```cmd
:: 设置 vcpkg 使用指定工具链
set VCPKG_PLATFORM_TOOLSET_VERSION=14.38

:: 设置 Unreal Engine 路径（根据实际安装位置调整）
set UNREAL_ENGINE_ROOT=C:\Program Files\Epic Games\UE_5.5

:: 进入 extern 目录
cd /d D:\data\onsite\OnsiteTemplate\Plugins\cesium-unreal\extern
```

3. 配置 CMake（指定工具链版本）：

```cmd
cmake -B build -S . -G "Visual Studio 17 2022" -A x64 -T "v143,version=14.38.33130"
```

4. 构建 Release 版本：

```cmd
cmake --build build --config Release --target install
```

5. 构建 Debug 版本（可选，用于调试）：

```cmd
cmake --build build --config Debug --target install
```

### 方法二：使用 PowerShell

```powershell
# 设置环境变量
$env:VCPKG_PLATFORM_TOOLSET_VERSION = "14.38"
$env:VCToolsVersion = "14.38.33130"
$env:CI = "1"  # 强制 vcpkg 使用指定的 MSVC 版本
$env:UNREAL_ENGINE_ROOT = "C:\Program Files\Epic Games\UE_5.5"
$env:EZVCPKG_BASEDIR = "D:\vcpkg-cache"  # 避免在驱动器根目录创建文件夹

# 进入 extern 目录
cd D:\data\onsite\OnsiteTemplate\Plugins\cesium-unreal\extern

# 配置（指定工具链版本）
cmake -B build -S . -G "Visual Studio 17 2022" -A x64 -T "v143,version=14.38.33130"

# 构建 Release
cmake --build build --config Release --target install --parallel

# 构建 Debug（可选）
cmake --build build --config Debug --target install --parallel
```

### 方法三：使用 vcvarsall 指定编译器版本

```cmd
:: 初始化指定版本的编译环境
"C:\Program Files\Microsoft Visual Studio\2022\Professional\VC\Auxiliary\Build\vcvarsall.bat" x64 -vcvars_ver=14.38

:: 设置环境变量
set VCPKG_PLATFORM_TOOLSET_VERSION=14.38
set VCToolsVersion=14.38.33130
set CI=1
set UNREAL_ENGINE_ROOT=C:\Program Files\Epic Games\UE_5.5

:: 进入目录并构建
cd /d D:\data\onsite\OnsiteTemplate\Plugins\cesium-unreal\extern
cmake -B build -S . -G "Visual Studio 17 2022" -A x64 -T "v143,version=14.38.33130"
cmake --build build --config Release --target install
```

## 一键构建脚本

创建 `build-cesium-native.bat` 脚本：

```batch
@echo off
setlocal

:: 设置路径（根据实际情况修改）
set PLUGIN_DIR=D:\data\onsite\OnsiteTemplate\Plugins\cesium-unreal
set UNREAL_ENGINE_ROOT=C:\Program Files\Epic Games\UE_5.5

:: 设置工具链版本
set VCPKG_PLATFORM_TOOLSET_VERSION=14.38
set VCToolsVersion=14.38.33130
set CI=1

:: 进入 extern 目录
cd /d "%PLUGIN_DIR%\extern"

:: 清理旧的构建目录（可选，取消注释启用）
:: if exist build rmdir /s /q build

:: 配置 CMake
echo 正在配置 CMake...
cmake -B build -S . -G "Visual Studio 17 2022" -A x64 -T "v143,version=14.38.33130" -DUNREAL_ENGINE_ROOT="%UNREAL_ENGINE_ROOT%"
if errorlevel 1 (
    echo CMake 配置失败！
    pause
    exit /b 1
)

:: 构建 Release 版本
echo 正在构建 Release 版本...
cmake --build build --config Release --target install --parallel
if errorlevel 1 (
    echo Release 构建失败！
    pause
    exit /b 1
)

:: 构建 Debug 版本（可选）
echo 正在构建 Debug 版本...
cmake --build build --config Debug --target install --parallel
if errorlevel 1 (
    echo Debug 构建失败！
    pause
    exit /b 1
)

echo.
echo ========================================
echo 构建完成！
echo ========================================
pause
```

## 生成 Visual Studio 项目文件

在项目根目录右键点击 `OnsiteTemplate.uproject`，选择 **Generate Visual Studio project files**。

或使用命令行：

```powershell
cd D:\data\onsite\OnsiteTemplate
& "C:\Program Files\Epic Games\UE_5.5\Engine\Build\BatchFiles\GenerateProjectFiles.bat" -project="$PWD\OnsiteTemplate.uproject" -game
```

## 解决方案配置说明

| Cesium for Unreal 配置 | cesium-native 配置 |
|------------------------|-------------------|
| Development Editor     | Release           |
| DebugGame Editor       | Debug (优先) 或 Release |

- **Development Editor**: 日常开发推荐，性能较好
- **DebugGame Editor**: 需要调试时使用，可以进入 cesium-native 源码调试

## 常见问题

### 网络问题 / GitHub 无法连接

如果遇到 `fatal: unable to access 'https://github.com/...'` 错误：

1. **配置 Git 代理**（如果有代理服务器）：

```powershell
# 设置代理（根据实际代理地址修改）
git config --global http.proxy http://127.0.0.1:8902
git config --global https.proxy http://127.0.0.1:8902

# 取消代理
git config --global --unset http.proxy
git config --global --unset https.proxy
```

2. **设置 EZVCPKG_BASEDIR 环境变量**（避免在驱动器根目录创建文件夹）：

```powershell
$env:EZVCPKG_BASEDIR = "D:\vcpkg-cache"
```

3. **手动预先克隆 vcpkg**（如果自动克隆失败）：

```powershell
# 创建缓存目录
New-Item -ItemType Directory -Path "D:\vcpkg-cache" -Force
cd D:\vcpkg-cache

# 克隆 vcpkg（版本号根据错误信息中显示的 commit 调整）
git clone --depth 1 --branch 2025.09.17 https://github.com/microsoft/vcpkg.git 2025.09.17

# Bootstrap vcpkg
cd 2025.09.17
.\bootstrap-vcpkg.bat

# 然后设置环境变量并重新运行 CMake
$env:EZVCPKG_BASEDIR = "D:\vcpkg-cache"
```

### 编译器版本不匹配

如果看到链接错误，检查：

1. `BuildConfiguration.xml` 中的 `CompilerVersion` 设置是否正确
2. 环境变量 `VCPKG_PLATFORM_TOOLSET_VERSION` 和 `VCToolsVersion` 是否设置
3. 环境变量 `CI=1` 是否设置（强制 vcpkg 使用指定版本）
4. CMake 配置时是否使用了 `-T "v143,version=14.38.33130"` 参数

### 强制重新构建 vcpkg 依赖

如果更改了工具链版本，需要清理 vcpkg 缓存：

```powershell
# 删除 ezvcpkg 目录
Remove-Item -Recurse -Force "C:\.ezvcpkg" -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force "$env:USERPROFILE\.ezvcpkg" -ErrorAction SilentlyContinue

# 删除 vcpkg 二进制缓存
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\vcpkg\archives" -ErrorAction SilentlyContinue

# 删除构建目录
Remove-Item -Recurse -Force "D:\data\onsite\OnsiteTemplate\Plugins\cesium-unreal\extern\build" -ErrorAction SilentlyContinue
```

### 查看实际使用的编译器版本

构建开始时会显示类似信息：

```
Using Visual Studio 2022 14.38.xxxxx toolchain (C:\Program Files\Microsoft Visual Studio\2022\...\VC\Tools\MSVC\14.38.33130)
```

确认版本号为 14.38.x 即可。

### 查找已安装的 MSVC 版本

```powershell
Get-ChildItem "C:\Program Files\Microsoft Visual Studio\2022\*\VC\Tools\MSVC" -Directory | Select-Object Name
```

## Visual Studio Code 配置（可选）

如果使用 VS Code 开发，可以配置自定义 CMake Kit。编辑 CMake Kits（Ctrl+Shift+P → CMake: Edit User-Local CMake Kits）：

```json
{
    "name": "14.38 - Visual Studio 2022 - x64 (UE 5.5)",
    "visualStudio": "your-vs-instance-id",
    "visualStudioArchitecture": "x64",
    "preferredGenerator": {
        "name": "Visual Studio 17 2022",
        "platform": "x64",
        "toolset": "host=x64,version=14.38"
    },
    "environmentVariables": {
        "VCToolsVersion": "14.38.33130",
        "VCPKG_PLATFORM_TOOLSET_VERSION": "14.38",
        "CI": "1",
        "UNREAL_ENGINE_ROOT": "C:\\Program Files\\Epic Games\\UE_5.5"
    }
}
```

## 快速参考

```powershell
# 完整构建命令（复制即用）
$env:VCPKG_PLATFORM_TOOLSET_VERSION = "14.38"
$env:VCToolsVersion = "14.38.33130"
$env:CI = "1"
$env:UNREAL_ENGINE_ROOT = "C:\Program Files\Epic Games\UE_5.5"
$env:EZVCPKG_BASEDIR = "D:\vcpkg-cache"
cd D:\data\onsite\OnsiteTemplate\Plugins\cesium-unreal\extern
cmake -B build -S . -G "Visual Studio 17 2022" -A x64 -T "v143,version=14.38.33130"
cmake --build build --config Release --target install --parallel
```


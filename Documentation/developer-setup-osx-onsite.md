# OnsiteTemplate 项目 - macOS 开发环境设置

针对 OnsiteTemplate 项目的 Cesium for Unreal 开发环境设置指南。

## 项目路径说明

```
项目根目录: /Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate
Cesium 插件: /Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal
cesium-native: /Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal/extern/cesium-native
Unreal Engine: 5.5
```

## 前置条件

- 安装 CMake (3.15 或更高版本): https://cmake.org/install/
- 安装 Xcode 14.1+: https://developer.apple.com/xcode/resources/
- (可选) 安装 [nasm](https://www.nasm.us/) 以获得更好的 JPEG 解码性能
- 安装 Unreal Engine 5.5

检查支持的 Xcode 版本：
```bash
cat "/Users/Shared/Epic Games/UE_5.5/Engine/Config/Apple/Apple_SDK.json"
```

## 更新 cesium-native 子模块

如果 `extern/cesium-native` 目录为空或缺失，运行：

```bash
cd /Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal
git submodule update --init --recursive
```

## 构建 cesium-native

### Debug 版本（开发调试）

```bash
export UNREAL_ENGINE_ROOT='/Users/Shared/Epic Games/UE_5.5'
cd /Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal/extern
cmake -B build -S . -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target install --parallel 14
```

### Release 版本（推荐用于日常开发）

```bash
export UNREAL_ENGINE_ROOT='/Users/Shared/Epic Games/UE_5.5'
cd /Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal/extern
cmake -B build -S . -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build --target install --parallel 14
```

### 创建符号链接

构建完成后，库文件会安装到 `Source/ThirdParty/lib/` 下对应的目录：

- `Darwin-arm64-Debug` - Apple Silicon Debug
- `Darwin-arm64-Release` - Apple Silicon Release
- `Darwin-x64-Debug` - Intel Debug  
- `Darwin-x64-Release` - Intel Release

Cesium for Unreal 期望在以下位置找到库：

- `Darwin-universal-Debug` - DebugGame 配置
- `Darwin-universal-Release` - Development/Shipping 配置

**对于仅在 Apple Silicon Mac 上开发**，创建符号链接：

```bash
cd /Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal/Source/ThirdParty/lib
ln -s ./Darwin-arm64-Debug Darwin-universal-Debug
ln -s ./Darwin-arm64-Release Darwin-universal-Release
```

**对于仅在 Intel Mac 上开发**，创建符号链接：

```bash
cd /Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal/Source/ThirdParty/lib
ln -s ./Darwin-x64-Debug Darwin-universal-Debug
ln -s ./Darwin-x64-Release Darwin-universal-Release
```

## 构建通用二进制（支持两种架构）

如需同时支持 Intel 和 Apple Silicon：

```bash
cd /Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal/extern

# 构建 Intel 版本
cmake -B build-x64 -S . -DCMAKE_OSX_ARCHITECTURES=x86_64 -DCMAKE_SYSTEM_NAME=Darwin -DCMAKE_SYSTEM_PROCESSOR=x86_64 -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build-x64 --target install --parallel 14

# 使用 lipo 创建通用库
mkdir -p /Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal/Source/ThirdParty/lib/Darwin-universal-Release

for f in /Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal/Source/ThirdParty/lib/Darwin-x86_64-Release/*.a
do
  arm64f=/Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal/Source/ThirdParty/lib/Darwin-arm64-Release/$(basename -- $f)
  x64f=/Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal/Source/ThirdParty/lib/Darwin-x86_64-Release/$(basename -- $f)
  universalf=/Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal/Source/ThirdParty/lib/Darwin-universal-Release/$(basename -- $f)
  if diff $arm64f $x64f; then
    cp $arm64f $universalf
  else
    lipo -create -output $universalf $arm64f $x64f
  fi
done
```

## 构建 iOS 版本

> **注意**: 需要先完成 macOS 版本的构建，否则 Unreal Editor 无法启动。

```bash
export UNREAL_ENGINE_ROOT='/Users/Shared/Epic Games/UE_5.5'
cd /Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal/extern
cmake -B build-ios -S . -GXcode -DCMAKE_TOOLCHAIN_FILE="unreal-ios-toolchain.cmake" -DCMAKE_BUILD_TYPE=Release
cmake --build build-ios --target install --config Release --parallel 14
```

## 生成 Xcode 项目文件

由于 OnsiteTemplate 已经是 C++ 项目，可以直接生成项目文件：

```bash
cd /Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate
"/Users/Shared/Epic Games/UE_5.5/Engine/Build/BatchFiles/Mac/GenerateProjectFiles.sh" -game -project="$PWD/OnsiteTemplate.uproject"
```

成功后会生成 `OnsiteTemplate (Mac).xcworkspace` 文件。

## 在 Xcode 中构建和运行

1. 双击 `OnsiteTemplate (Mac).xcworkspace` 打开 Xcode
2. 在 **Product → Scheme** 菜单中选择 `OnsiteTemplateEditor`
3. 如需 Debug 配置：**Product → Scheme → Edit Scheme → Run → Info**，将 "Build Configuration" 改为 "DebugGame"
4. 构建：**Product → Build**
5. 运行：**Product → Run** 启动 Unreal Editor

## 常见问题

### Xcode 配置问题

如果看到以下错误：
> Your Mac is set to use CommandLineTools for its build tools

运行：
```bash
sudo xcode-select -s /Applications/Xcode.app
```

### SDK 版本问题

如果看到：
> Platform Mac is not a valid platform to build

检查 Xcode 版本是否受支持：
```bash
cat "/Users/Shared/Epic Games/UE_5.5/Engine/Config/Apple/Apple_SDK.json"
```

## 快速构建脚本

可以创建一个 shell 脚本简化日常构建：

```bash
#!/bin/bash
# build-cesium-native.sh

export UNREAL_ENGINE_ROOT='/Users/Shared/Epic Games/UE_5.5'
PLUGIN_DIR="/Users/hexi/Documents/UEProjects/onsite/OnsiteTemplate/Plugins/cesium-unreal"

cd "$PLUGIN_DIR/extern"

# 构建 Release 版本
cmake -B build -S . -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build --target install --parallel 14

# 创建符号链接（Apple Silicon）
cd "$PLUGIN_DIR/Source/ThirdParty/lib"
ln -sf ./Darwin-arm64-Release Darwin-universal-Release
ln -sf ./Darwin-arm64-Debug Darwin-universal-Debug

echo "Build complete!"
```

保存为 `build-cesium-native.sh` 并赋予执行权限：
```bash
chmod +x build-cesium-native.sh
```


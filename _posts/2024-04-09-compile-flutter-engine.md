---
layout: single
title:  "编译 Flutter 引擎"
date:   2024-04-09 01:20:00 +0800
categories: mobile_devs
---

官方 wiki: [compiling the engine](https://github.com/flutter/flutter/wiki/Compiling-the-engine)
## 拉取源码

1. fork Flutter engine 仓库：https://github.com/flutter/engine

2. clone depot_tools
   depot_tools 包含了 `gclient`, `repo` 等命令行工具，用于拉取 Flutter engine, Chromium 等项目的构建依赖

   ```shell
   git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
   export PATH=$PATH:<Current Path>/depot_tools # set up environment variable
   ```

3. 创建引擎存放目录，添加 \.gclient 配置文件

   ```shell
   mkdir engine
   cd engine
   touch .gclient
   ```

   往 .gclient 中添加如下配置内容：

   ```json
   solutions = [
     {
       "managed": False,
       "name": "src/flutter",
       "url": "git@github.com:<GITHUB_USER_NAME>/engine.git",
       "custom_deps": {},
       "deps_file": "DEPS",
       "safesync_url": "",
     },
   ]
   ```

   其中，name 指定了源码下载的相对路径(engine/src/flutter)

4. 同步代码

   ```shell
   gclient sync
   ```

   过程可能会较漫长，使用 `gclient sync` 拉取 engine 源码需要较长时间，若想加快速度可先切换到 src/flutter 中使用 git 拉取 fork 代码：

   ```shell
   cd src/flutter
   git init
   git remote add origin git@github.com:<GITHUB_USER_NAME>/engine.git
   git pull origin main
   ```

   之后回切回 engine 目录再执行 `gclient sync` 即可

   注意：`gclient sync` 同步代码后进度显示 100%，但此时依然会下载依赖，请勿随意终止

5. 关联官方仓库

   ```shell
   cd src/flutter
   git remote add upstream git@github.com:flutter/engine.git
   ```

   之后拉取官方仓库代码可直接使用 `git pull upstream`

## 切换源码版本

通过命令行获取当前安装 flutter engine(3.19.0) 的 commit id:

![Get flutter engine version](/assets/compile-flutter-engine/get_flutter_engine_version.jpg)

切换到对应 commit:

![Switch to corresponding commit](/assets/compile-flutter-engine/reset_commit.jpg)

再执行以下命令同步依赖

```shell
gclient sync -D --with_branch_heads --with_tags -v
```

## 编译源码

### 创建构建脚本

切换到 src 目录下执行以下命令可以创建对应的构建脚本：

```shell
# For Android
./flutter/tools/gn --android --unoptimized
./flutter/tools/gn --android --android-cpu arm64 --unoptimized # only for newer 64-bit Android device

# For iOS
./flutter/tools/gn --ios --unoptimized
./flutter/tools/gn --ios --unoptimized --simulator-cpu=arm64 # for arm64 simulator
```

更多的构建脚本可以参照官方 wiki

注意，后续的 engine 调试还需要依赖于宿主机 engine，所以还需要创建一份宿主机的 engine 构建脚本：

```shell
./flutter/tools/gn --unoptimized # Generate out/host_debug_unopt
```

注意，如果使用的是 Apple silicon Mac，则推荐执行以下命令创建对应的脚本：

```shell
./flutter/tools/gn --unoptimized --mac-cpu arm64 # Generate out/host_debug_unopt_arm64
```

### 构建引擎

使用 `ninja` 进行编译构建：

```shell
ninja -C out/android_debug_unopt
ninja -C out/android_debug_unopt_arm64

ninja -C out/host_debug_unopt # or ninja -C out/host_debug_unopt_arm64
```

首次编译耗时较长，之后改动引擎重新构建则会采用增量编译

注：在构建 Android engine 的时候可能最后会卡在 test/scenario_app 的 lint 阶段，这时候可以终止构建，因为引擎已经构建完成

## 调试引擎

### 配置调试参数

在 flutter 项目构建时配置如下参数使用自编译的 engine（以 Android 为例）：

```shell
flutter run --local-engine-src-path <Your engine path>/src --local-engine=android_debug_unopt_arm64 --local-engine-host=host_debug_unopt
```

也可自行将构建参数添加到 Android Studio 中的 run configuation 中

### 验证是否成功应用自编译 engine

打开文件 engine/src/flutter/shell/common/engine.cc，在 engine 的 `Run` 函数中添加以下代码并重新编译引擎：

![Edit engine.cc](/assets/compile-flutter-engine/edit_engine_cc.jpg)

之后在 AS 中运行自定义引擎构建的 flutter 应用时，可看到如下日志输出，即证明成功应用自编译 engine：

![Engine log output](/assets/compile-flutter-engine/engine_log_output.jpg)

### 来点小小的“魔改”

在 Flutter 中，文字应用 `TextOverflow.ellipsis` 时默认的截断样式是 …，接下来我们尝试将三个点替换为三个 \+ 号。

打开文件 engine/src/flutter/third_party/skia/modules/skparagraph/include/ParagraphStyle.h，找到 `getEllipsis` 函数，对其进行简单修改：

![Edit getEllipsis](/assets/compile-flutter-engine/edit_getEllipsis.png)

重新编译引擎并运行 app，可以发现截断样式被替换为了 \+ 号

<img src="/assets/compile-flutter-engine/modified_ellipsis.jpg" alt="Modified ellipsized effect" style="zoom: 50%;" />

### 断点调试引擎

参阅 [Debugging the engine](https://github.com/flutter/flutter/wiki/Debugging-the-engine)
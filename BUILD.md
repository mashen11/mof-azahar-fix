# 构建

基线：`azahar-emu/azahar` 的提交 `219d743`（对应上游 tag `2126.2` 之后的提交）。
补丁集：`patches/0001-mirror-of-fate-motion-fix.patch`。

## 取源码并打补丁

```bash
git clone https://github.com/azahar-emu/azahar.git
cd azahar
git checkout 219d743
git am /path/to/0001-mirror-of-fate-motion-fix.patch
```

补丁会修改 9 个文件：

```
src/core/savestate.cpp
src/core/perf_stats.cpp
src/core/render_census.h          (新增)
src/video_core/gpu.cpp
src/video_core/gpu_impl.h
src/video_core/pica/pica_core.cpp
src/video_core/pica/pica_core.h
src/video_core/pica/pica_trace.h  (新增)
src/common/settings.h
src/android/app/src/main/jni/config.cpp
```

> GPLv2 合规：完整的对应源码 = 上面这条 `git checkout` 的上游提交 + 本目录的补丁。两者合起来
> 可以完整还原出发布二进制所用的代码状态。

## Windows 桌面版

依赖：Visual Studio 2022（MSVC x64）、Qt 6.8、Vulkan SDK、CMake + Ninja。

```bash
cmake -G Ninja -B build -DCMAKE_BUILD_TYPE=Release -DENABLE_QT=ON -DENABLE_VULKAN=ON
ninja -C build azahar.exe
```

产物在 `build/bin/Release/`，需要连同同目录的 Qt DLL 一起分发。

## 掌机版（Android）

依赖：Android SDK、NDK r27d、JDK 21。**必须在 ext4 分区内构建**（`/mnt/c` 下会因 `aapt2` 缺少
执行权限而失败）。

```bash
export ANDROID_HOME=$HOME/android-sdk-full
export ANDROID_NDK=/opt/android/ndk/android-ndk-r27d
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
export AZAHAR_SKIP_VVL=1        # 跳过 Vulkan 验证层，release 不需要，且它只有 GitHub 一个下载源

cd src/android
./gradlew assembleVanillaRelWithDebInfo
```

产物：`src/android/app/build/outputs/apk/vanilla/relWithDebInfo/app-vanilla-relWithDebInfo.apk`

`AZAHAR_SKIP_VVL=1` 是必要的：构建脚本会去 GitHub 拉 Vulkan 验证层的预编译包，国内网络通常
拿不到，而 release 构建不需要它。

## 运行时参数

编译期默认值（改代码才生效，Android 上没有环境变量）：

| 参数 | 默认 | 位置 |
|---|---|---|
| `DelayScale` | 1.6 | `pica_core.h` |
| `MinSceneDelayNs` | 15 ms | `pica_core.h` |
| `simulate_3ds_gpu_timings` | true | `common/settings.h` |
| `frame_limit` | 100 | `run_mof.bat` / 游戏属性 |

桌面端可以用环境变量临时覆盖，方便做 A/B：

```bash
AZAHAR_GPU_DELAY_SCALE=1.0      # 1.0 = 复现缺陷
AZAHAR_MIN_SCENE_DELAY_US=0     # 关闭场景下限
AZAHAR_MIN_SCENE_DELAY_US=16500 # 更激进，帧率会掉
```

## 诊断插桩

`render_census.h` 会每 240 个呈现帧往日志打一行 `RENDERSTATS` / `RENDERSHIST`：

```
RENDERSTATS frames=240 char_avg=2.88 poses_per_frame=0.958 renders_per_frame=1.000 ...
RENDERSHIST char_draws[0:10 3:230 ] poses[0:10 1:230 ] fills_per_frame=8.62
```

判读标准：

- `char_draws[3:N]` 且 `poses[1:N]` —— 正常，一帧画角色 3 次（3 个渲染目标）、1 个姿态；
- 出现 `6:M` 且 `poses[...,2:M]` —— 双遍渲染，也就是缺陷帧；
- `renders_per_frame` 在 `targets_per_frame` 不足 3 时**不可信**（分母会动），要看直方图。

日志位置：Windows `%APPDATA%\azahar\log\azahar_log.txt`，Android `adb logcat | grep RENDERSTATS`。

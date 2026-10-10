# RustDesk 客户端编译环境指南（Windows 端 + Android 端）

> 核对时间：2026-10-08。依据 rustdesk/rustdesk 仓库 master 分支（提交 `25d2c6d`，2026-10-08）的 `.github/workflows/flutter-build.yml`、`.github/workflows/bridge.yml`、`vcpkg.json`、`build.py`、`flutter/build_android_deps.sh`、`flutter/ndk_arm64.sh`。版本号全部取自官方 CI，不是凭记忆写的。官方 CI 会随时升级，开始搭环境前请再看一眼 `flutter-build.yml` 顶部的 `env:` 段。

## 0. 官方 CI 当前使用的版本

| 组件 | 版本 | 出处 |
|---|---|---|
| RustDesk | 1.5.0 | `Cargo.toml` / CI `VERSION` |
| Rust 工具链 | **1.75**（Windows、Android 都是） | `RUST_VERSION`、`SCITER_RUST_VERSION`；注释说明 1.78 有 ABI 变化，不要用新版 |
| Flutter（Windows x64 / Android） | **3.24.5** stable，且要打官方补丁 | `FLUTTER_VERSION`、`ANDROID_FLUTTER_VERSION` |
| Flutter（仅生成 bridge 代码用） | 3.22.3 | `bridge.yml` |
| flutter_rust_bridge_codegen | 1.80.1（`--features uuid`） | `bridge.yml` |
| cargo-expand | 1.0.95 | `bridge.yml` |
| LLVM / Clang | 15.0.6 | `LLVM_VERSION` |
| vcpkg | 提交 `9e593bb18ea69cc5095e012465dcd675a822ed0d`（2026.07.29） | `VCPKG_COMMIT_ID` |
| vcpkg 三元组（Windows x64） | `x64-windows-static` | 构建矩阵 |
| Android NDK | **r28c** | `NDK_VERSION` |
| cargo-ndk | 3.1.2 | `CARGO_NDK_VERSION` |
| JDK | 17 | Android 构建步骤 |
| Android SDK | compileSdk / targetSdk 36，minSdk 22 | `flutter/android/app/build.gradle` |
| Windows 构建机 | windows-2022（Visual Studio 2022） | 构建矩阵 |
| Android 构建机 | Ubuntu（官方 Android 构建只在 Linux 上跑） | 构建矩阵 |

vcpkg 会装的依赖（来自 `vcpkg.json`）：aom、libjpeg-turbo、opus、libvpx、libyuv、mfx-dispatch、ffmpeg 等，Android 额外有 cpu-features。第一次编译 vcpkg 依赖要 30 分钟到 1 小时以上，属正常现象。

## 1. 推荐路线：先用 GitHub Actions 云端编译

本地搭环境最容易出问题的是版本对不齐（Rust 1.75、Flutter 3.24.5 + 补丁、自定义 Flutter 引擎、bridge 代码生成）。官方 CI 把这些全部自动化了，Fork 之后可以直接复用：

1. 在 GitHub 上 Fork `rustdesk/rustdesk`。
2. 打开 Fork 仓库的 **Actions** 页，点“I understand my workflows, go ahead and enable them”。
3. 选 **Flutter Nightly Build** 工作流，点 **Run workflow** 手动触发（它是 `workflow_dispatch`，会调用 `flutter-build.yml` 编译所有平台）。
4. 跑完后在 Fork 仓库的 Releases 里会出现 `nightly` 预发布版本，里面有 Windows 安装包和 Android APK。

注意：

- 没有配置签名密钥时，Android APK 用的是 debug 签名（CI 里 `sed` 把 release 签名换成了 debug），可以安装测试，但正式分发前要配自己的签名（见 3.6）。
- 这个工作流会编译 macOS、iOS、Linux 等全部平台，比较慢且耗 Actions 分钟数。公共仓库的 Actions 免费；后续我可以在你的 Fork 里裁剪出一个“只编 Windows x64 + Android arm64”的工作流。
- Nightly 工作流里有一个每天定时触发的 `schedule`，Fork 后建议去掉，避免每天白跑。

**本地环境仍然需要**：改界面时要频繁调试，靠 CI 太慢。下面是本地搭建步骤。

## 2. Windows 端（本地，x64）

### 2.1 安装基础工具

1. **Visual Studio 2022**（Community 或 Build Tools 均可），勾选“使用 C++ 的桌面开发”工作负载（含 MSVC、Windows SDK、CMake）。
2. **Git for Windows**。
3. **Python 3**（`build.py` 是 Python 脚本），安装时勾选“Add to PATH”。
4. **LLVM 15.0.6**：从 LLVM 的 GitHub Releases 下载 `LLVM-15.0.6-win64.exe` 安装，然后设置环境变量：
   ```powershell
   setx LIBCLANG_PATH "C:\Program Files\LLVM\bin"
   ```
   （bindgen 需要 libclang，版本不对可能编译报错。）

### 2.2 Rust 1.75

```powershell
# 安装 rustup（https://rustup.rs），然后：
rustup toolchain install 1.75 --target x86_64-pc-windows-msvc --component rustfmt
# 在仓库根目录固定工具链，避免用到新版：
rustup override set 1.75
```

### 2.3 vcpkg（固定到官方提交）

```powershell
git clone https://github.com/microsoft/vcpkg C:\vcpkg
cd C:\vcpkg
git checkout 9e593bb18ea69cc5095e012465dcd675a822ed0d
.\bootstrap-vcpkg.bat
setx VCPKG_ROOT "C:\vcpkg"
```

重开一个终端，在 **rustdesk 仓库根目录**（有 `vcpkg.json` 的地方）执行：

```powershell
$env:VCPKG_DEFAULT_HOST_TRIPLET="x64-windows-static"
C:\vcpkg\vcpkg install --triplet x64-windows-static --x-install-root="C:\vcpkg\installed"
```

### 2.4 Flutter 3.24.5 + 官方补丁 + 自定义引擎

```powershell
git clone https://github.com/flutter/flutter.git -b 3.24.5 C:\flutter
setx PATH "$env:PATH;C:\flutter\bin"
# 重开终端
flutter doctor -v
flutter precache --windows
```

官方 CI 对 Flutter 做了两处修改，本地也要做：

1. **打下拉菜单补丁**（在 Flutter 安装目录里执行）：
   ```powershell
   cd C:\flutter
   git apply <rustdesk仓库路径>\.github\patches\flutter_3.24.4_dropdown_menu_enableFilter.diff
   ```
2. **替换为 RustDesk 自定义的 Windows 引擎**（对应上游 issue flutter/flutter#155685）：
   ```powershell
   Invoke-WebRequest -Uri https://github.com/rustdesk/engine/releases/download/main/windows-x64-release.zip -OutFile windows-x64-release.zip
   Expand-Archive windows-x64-release.zip -DestinationPath windows-x64-release
   Move-Item -Force windows-x64-release\* C:\flutter\bin\cache\artifacts\engine\windows-x64-release\
   ```

### 2.5 生成 Rust ↔ Dart 桥接代码

官方 CI 用 Flutter 3.22.3 + flutter_rust_bridge_codegen 1.80.1 在 Linux 上生成，然后把生成物（`flutter/lib/generated_bridge.dart`、`*.freezed.dart`、`bridge_generated.h` 等）交给各平台编译。最省事的做法：

- **做法 A（推荐）**：在 WSL2 的 Ubuntu 里生成（与第 3 节 Android 环境共用），生成后的文件在同一个仓库目录里，Windows 侧直接用。
- **做法 B**：在 Windows 上直接生成：
  ```powershell
  cargo install cargo-expand --version 1.0.95 --locked
  cargo install flutter_rust_bridge_codegen --version 1.80.1 --features "uuid" --locked
  cd flutter; flutter pub get; cd ..
  flutter_rust_bridge_codegen --rust-input ./src/flutter_ffi.rs --dart-output ./flutter/lib/generated_bridge.dart --c-output ./flutter/macos/Runner/bridge_generated.h
  ```
  官方生成时用的是 Flutter 3.22.3，并把 `flutter/pubspec.yaml` 里的 `extended_text: 14.0.0` 临时改成 `13.0.0`。用 3.24.5 直接生成通常也能用，如果出现 freezed 相关编译错误，就按官方做法切到 3.22.3 生成。

**只要改了 `src/flutter_ffi.rs` 里的接口，就要重新生成一次。** 只改 Dart 界面不需要。

### 2.6 编译

```powershell
# 本地调试：
python build.py --flutter

# 与官方发布一致（便携版 + 硬件编码 + 显存直通编码）：
python build.py --portable --flutter --skip-portable-pack --hwcodec --vram
```

产物在 `flutter\build\windows\x64\runner\Release\`。

只改 Dart 界面时，可以先 `python build.py --flutter` 编一次 Rust 库，之后在 `flutter` 目录用 `flutter run -d windows` 热重载调界面，快很多。

## 3. Android 端（推荐在 WSL2 Ubuntu 22.04/24.04 中编译）

官方 Android 只在 Linux 上构建，`build_android_deps.sh` 和 `ndk_*.sh` 都是 bash 脚本，所以 Windows 电脑上请用 WSL2。

### 3.1 系统依赖

```bash
sudo apt-get update
sudo apt-get install -y clang cmake curl git g++ gcc-multilib g++-multilib \
  libclang-dev llvm-dev nasm ninja-build pkg-config wget unzip zip \
  openjdk-17-jdk-headless
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
```

### 3.2 Android SDK 与 NDK r28c

```bash
mkdir -p ~/android-sdk/cmdline-tools && cd ~/android-sdk/cmdline-tools
# 从 developer.android.com 下载 commandlinetools-linux-*.zip 解压，并把目录改名为 latest
export ANDROID_HOME=~/android-sdk
export PATH=$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH
sdkmanager "platform-tools" "platforms;android-36" "build-tools;36.0.0"

# NDK r28c：从 https://developer.android.com/ndk/downloads 下载 android-ndk-r28c-linux.zip
unzip android-ndk-r28c-linux.zip -d ~/
export ANDROID_NDK_HOME=~/android-ndk-r28c
export ANDROID_NDK_ROOT=$ANDROID_NDK_HOME
```

（建议把这些 `export` 写进 `~/.bashrc`。）

### 3.3 Rust 1.75 + cargo-ndk

```bash
curl https://sh.rustup.rs -sSf | sh
rustup toolchain install 1.75 --component rustfmt
rustup default 1.75
rustup target add aarch64-linux-android      # 主流手机 arm64-v8a
cargo install cargo-ndk --version 3.1.2 --locked
```

### 3.4 vcpkg 与 Android 依赖

```bash
git clone https://github.com/microsoft/vcpkg ~/vcpkg
cd ~/vcpkg && git checkout 9e593bb18ea69cc5095e012465dcd675a822ed0d && ./bootstrap-vcpkg.sh
export VCPKG_ROOT=~/vcpkg

cd <rustdesk 仓库>
./flutter/build_android_deps.sh arm64-v8a
```

### 3.5 Flutter 3.24.5 + 补丁

```bash
git clone https://github.com/flutter/flutter.git -b 3.24.5 ~/flutter
export PATH=~/flutter/bin:$PATH
cd ~/flutter && git apply <rustdesk 仓库>/.github/patches/flutter_3.24.4_dropdown_menu_enableFilter.diff
flutter doctor -v
```

桥接代码按 2.5 生成（同一个仓库只需生成一次）。

### 3.6 编译 APK（arm64）

```bash
cd <rustdesk 仓库>

# 1) 编译 Rust 库（等同 flutter/ndk_arm64.sh）
cargo ndk --platform 21 --target aarch64-linux-android build --locked --release --features flutter,hwcodec

# 2) 放入 jniLibs
mkdir -p flutter/android/app/src/main/jniLibs/arm64-v8a
cp target/aarch64-linux-android/release/liblibrustdesk.so \
   flutter/android/app/src/main/jniLibs/arm64-v8a/librustdesk.so
cp $ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/aarch64-linux-android/libc++_shared.so \
   flutter/android/app/src/main/jniLibs/arm64-v8a/

# 3) 打包
cd flutter
flutter build apk --release --target-platform android-arm64 --split-per-abi
# 产物：flutter/build/app/outputs/flutter-apk/app-arm64-v8a-release.apk
```

签名：`flutter/android/app/build.gradle` 默认读取 release 签名配置。没有自己的签名时，官方 CI 是把 `signingConfigs.release` 替换成 `signingConfigs.debug` 来出包的。正式使用建议生成自己的 keystore：

```bash
keytool -genkey -v -keystore ~/remote-release.jks -keyalg RSA -keysize 2048 -validity 10000 -alias remote
```

然后在 `flutter/android/key.properties` 中配置（这个文件和 `.jks` 都不要提交到 GitHub）。

CI 签名：`remote-build.yml` 出包后会用 `r0adkll/sign-android-release` 重新签名，只要在仓库 Settings → Secrets and variables → Actions 里配好下面四个 secret，不用改工作流：

| Secret | 内容 |
|---|---|
| `ANDROID_SIGNING_KEY` | `.jks` 文件的 base64（`base64 -w0 remote-release.jks`） |
| `ANDROID_ALIAS` | 生成 keystore 时的 `-alias` |
| `ANDROID_KEY_STORE_PASSWORD` | keystore 密码 |
| `ANDROID_KEY_PASSWORD` | key 密码（keytool 默认与 keystore 密码相同） |

四个 secret 都没配时，CI 发布的是 debug 签名的 APK。keystore 要长期保存，换了 keystore 的 APK 在手机上必须先卸载旧版才能安装。

只改界面时，Rust 库编一次后，在 `flutter` 目录用 `flutter run`（手机开 USB 调试）热重载即可。

## 4. 仓库结构速览（Fork 后要改的地方）

以下基于官方仓库 master 的结构，Fork 后我会在你的仓库里再核对一遍。

| 想改什么 | 位置 |
|---|---|
| **默认 ID 服务器** | `libs/hbb_common/src/config.rs`：`pub const RENDEZVOUS_SERVERS: &[&str] = &["rs-ny.rustdesk.com"];` |
| **默认服务器公钥** | 同文件：`pub const RS_PUB_KEY: &str = "OeVuKk5n...";` 换成你的 `id_ed25519.pub` |
| 默认端口 | 同文件：`RENDEZVOUS_PORT = 21116`、`RELAY_PORT = 21117` |
| 应用名 | 显示名“星控”：Rust `src/common.rs` 的 `APP_DISPLAY_NAME`、Dart `flutter/lib/consts.dart` 的 `kAppDisplayName`、Android `AndroidManifest.xml` / `strings.xml`、Windows `Runner.rc`。内部名 `APP_NAME`（`libs/hbb_common/src/config.rs`）为 `"XingKong"`：它是 Windows 服务名、注册表键、安装目录、exe 名和配置目录名，只能用字母数字，改成中文会装不上 |
| 运行时自定义服务器 | 客户端设置项 `custom-rendezvous-server`、`relay-server`、`key`；Windows 还支持把 exe 命名为 `rustdesk-host=域名,key=公钥.exe` 自动带入（`src/custom_server.rs`） |
| Flutter 界面 | `flutter/lib/`：`mobile/pages/`（手机端：`home_page`、`connection_page`、`remote_page`、`server_page`、`settings_page` 等）、`desktop/pages/`（电脑端）、`common/`、`models/` |
| Rust ↔ Dart 接口 | `src/flutter_ffi.rs`（改了要重新生成桥接代码） |
| 隐私模式（后续阶段） | `src/privacy_mode.rs` 与 `src/privacy_mode/` |
| Android 包名 | `flutter/android/app/build.gradle`：`applicationId "com.starared.xingkong"`（已改；不能和官方 RustDesk 同名，否则手机安全扫描会按“签名不符的篡改版”报毒） |
| 构建脚本 | `build.py`（桌面）、`flutter/build_android_deps.sh`、`flutter/ndk_*.sh`（Android） |
| CI | `.github/workflows/flutter-build.yml` |

**重要：`libs/hbb_common` 是 git 子模块**（指向 `rustdesk/hbb_common`）。默认服务器地址和公钥在这个子模块里，所以改它有两种办法：

1. 同时 Fork `rustdesk/hbb_common`，在 `.gitmodules` 里把子模块地址改成你的 Fork（干净，便于以后同步上游）；
2. 或者不改常量，改为在 Flutter/Rust 启动时写入默认设置（`DEFAULT_SETTINGS` / `HARD_SETTINGS`，也在 `config.rs` 中），这样不用 Fork 子模块。

我倾向于方案 1，Fork 后由我来改。

## 5. 常见坑

| 现象 | 处理 |
|---|---|
| Rust 编译报 sciter / i128 布局相关错误 | Rust 版本不是 1.75。`rustup override set 1.75`。 |
| bindgen 找不到 libclang | 设置 `LIBCLANG_PATH`，并确认 LLVM 是 15.x。 |
| vcpkg 编译 ffmpeg 失败 | 确认 vcpkg 切到了官方提交；Windows 下确认 VS2022 C++ 工作负载完整；查看 `C:\vcpkg\buildtrees\ffmpeg\*.log`。 |
| Flutter 报 `DropdownMenu` / `enableFilter` 相关错误 | 没打 Flutter 补丁。 |
| Windows 窗口异常/渲染问题 | 没替换 RustDesk 自定义引擎（2.4 第 2 步）。 |
| Dart 报 `generated_bridge.dart` 找不到或 freezed 不匹配 | 没生成桥接代码，或生成时的 Flutter 版本不对，按 2.5 重新生成。 |
| APK 启动闪退 | 检查 `jniLibs/arm64-v8a/` 下是否同时有 `librustdesk.so` 和 `libc++_shared.so`。 |
| Gradle 内存不足 | `flutter/android/gradle.properties` 中把 `-Xmx1024M` 改为 `-Xmx2g`（官方 CI 也这么改）。 |

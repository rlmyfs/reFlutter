
---

[![stars](https://img.shields.io/github/stars/Impact-I/reFlutter)](https://github.com/Impact-I/reFlutter/stargazers)

<p align="center"><img src="https://user-images.githubusercontent.com/87244850/135659542-22bb8496-bf26-4e25-b7c1-ffd8fc0cea10.png" width="75%"/></p>

**更多内容请参见博客：** <https://swarm.ptsecurity.com/fork-bomb-for-flutter/>

这个框架用于帮助 Flutter 应用逆向工程，使用已经打补丁并准备好用于应用重新打包的 Flutter 库。该库修改了 snapshot 反序列化流程，以便你能够方便地进行动态分析。

主要特性：

- `socket.cc` 打了补丁用于流量监控和拦截；
- `dart.cc` 修改为打印类、函数和一些字段；
- 显示函数的绝对代码偏移；
- 包含一些成功编译所需的轻量修改；
- 如果你想实现自己的补丁，支持使用专门制作的 Dockerfile 进行手动 Flutter 代码修改。

### 支持的引擎

- Android: arm64, arm32；
- iOS: arm64；
- Release: Stable, Beta

### 安装

```
# Linux, Windows, MacOS
pip3 install reflutter==0.8.6
```

### 用法

```console
impact@f:~$ reflutter main.apk

Please enter your Burp Suite IP: <input_ip>

SnapshotHash: 8ee4ef7a67df9845fba331734198a953
The resulting apk file: ./release.RE.apk
Please sign the apk file

impact@f:~$ reflutter main.ipa
```

### 流量拦截

你需要指定与运行 Flutter 应用设备处于同一网络中的 Burp Suite 代理服务器 IP。然后在 `BurpSuite -> Listener Proxy -> Options` 选项卡中配置代理：

- 添加端口：`8083`
- 绑定地址：`All interfaces`
- 请求处理：启用 invisible proxying = `True`

<p align="center"><img src="https://user-images.githubusercontent.com/87244850/135753172-20489ef9-0759-432f-b2fa-220607e896b8.png" width="84%"/></p>

无需安装证书或获取 root 权限即可在 Android 上使用。reFlutter 还支持绕过某些 Flutter 证书固定实现。

> ⚠️ **注意：** 从 Flutter 版本 **3.24.0**（snapshot hash: `80a49c7111088100a233b2ae788e1f48`）开始，硬编码的代理 IP 和端口已被移除。你现在需要直接在设备上配置代理。

#### 在 Android 上

使用 ADB 配置设备代理：

```bash
adb -s <device> shell "settings put global http_proxy <proxy_ip:port>"
```

对 APK 进行签名和对齐。建议使用工具 [uber-apk-signer](https://github.com/patrickfav/uber-apk-signer/releases/tag/v1.2.1)。

运行应用程序。通过二分法确定 `_kDartIsolateSnapshotInstructions`。reFlutter 会将转储文件写入应用的根目录并设置 777 权限。使用以下命令取回：

```bash
adb -d shell "cat /data/data/<PACKAGE_NAME>/dump.dart" > dump.dart
```

<details>
<summary>文件内容</summary>

```dart
Library:'package:anyapp/navigation/DeepLinkImpl.dart' Class: Navigation extends Object {
String* DeepUrl = anyapp://evil.com/ ;
...
```

</details>

### 在 iOS 上使用

运行 `reflutter main.ipa` 后，在设备上执行应用。转储文件路径会打印到 Xcode 控制台日志中：

```
Current working dir: /private/var/mobile/Containers/Data/Application/<UUID>/dump.dart
```

从设备取出该文件。

<p align="center"><img src="https://user-images.githubusercontent.com/87244850/135860648-a13ba3fd-93d2-4eab-bd38-9aa775c3178f.png" width="100%"/></p>

### Frida

```
frida-tools==13.7.1
frida==16.7.19
```

在 Frida 脚本中使用转储偏移量：

```bash
frida -U -f <package> -l frida.js
```

要查找 `_kDartIsolateSnapshotInstructions`：

```bash
readelf -Ws libapp.so
```

查看 `Value` 字段。

### 待办事项

- [x] 显示函数的绝对代码偏移；
- [ ] 提取更多字符串和字段；
- [x] 添加 socket 补丁；
- [ ] 通过 Fork 和 Github Actions 扩展 Debug 引擎支持；
- [ ] 改进对 zip 存档内 `App.framework` 和 `libapp.so` 的检测

### 构建引擎

引擎使用基于 enginehash.csv 中数据的 [GitHub Actions](https://github.com/Impact-I/reFlutter/actions) 构建。快照哈希从以下地址获取：

```
https://storage.googleapis.com/flutter_infra_release/flutter/<hash>/android-arm64-release/linux-x64.zip
```

<details>
<summary>release</summary>

[![gif](https://user-images.githubusercontent.com/87244850/135758767-47b7d51f-8b6c-40b5-85aa-a13c5a94423a.gif)](https://github.com/Impact-I/reFlutter/actions)

</details>

### 自定义构建

手动 Flutter 代码补丁支持 Docker：

```bash
git clone https://github.com/Impact-I/reFlutter && cd reFlutter
docker build -t reflutter -f Dockerfile .
```

运行：

```bash
docker run -it -v "$(pwd):/t" -e HASH_PATCH=<Snapshot_Hash> -e COMMIT=<Engine_commit> reflutter
```

示例：

```bash
docker run -it -v "$(pwd):/t" -e HASH_PATCH=aa64af18e7d086041ac127cc4bc50c5e -e COMMIT=d44b5a94c976fbb65815374f61ab5392a220b084 reflutter
```

#### 示例：构建 Android ARM64（Linux/Windows）

```bash
docker run -e WAIT=300 -e x64=0 -e arm=0 -e HASH_PATCH=<Snapshot_Hash> -e COMMIT=<Engine_commit> --rm -iv${PWD}:/t reflutter
```

参数：

- `-e x64=0`：禁用 x64 构建
- `-e arm64=0`：禁用 arm64 构建
- `-e arm=0`：禁用 arm32 构建
- `-e WAIT=300`：修改源代码前等待的秒数
- `-e HASH_PATCH`：来自 enginehash.csv 的快照哈希
- `-e COMMIT`：引擎提交哈希

---

# 安装包
    pip install .

# 然后可以直接运行
    reflutter yangjibao_2.2.6.apk

# 文件签名
    uber-apk-signer.bat
    java -jar uber-apk-signer-1.2.1.jar --allowResign -a release.RE.apk
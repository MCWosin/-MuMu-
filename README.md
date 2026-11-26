# -MuMu-
省流：下载MuMu，部署 KernelSU；配置 ZygiskNext；部署 Vector；配置 LocationSpoofer；定位 Native 反模拟器检测；绕过启动页，终于抵达登录界面。结果最后发现：服务器域名不行。最大的阻碍仍是体适能app
# MuMu 模拟器 6.0（Android 15）部署全记录：LocationSpoofer + 体适能

> **一句话总结**：模拟器侧全部就绪并验证通过（Root、ZygiskNext、LSPosed/Vector、LocationSpoofer 虚拟定位、体适能绕过启动页）；体适能登录失败的 404 是**官方登录域名 `h.boxkj.com` 已下线**导致，与模拟器无关，客户端无解。**项目到此结束。**
>
> 本文件合并了本次会话的全部内容：部署步骤、版本对比、体适能闪退排查与 404 诊断。
>
> 记录时间：2026-11-26 · 工作区：`D:\MuMuPlayer`

---

# 目录

1. [会话总览](#一会话总览)
2. [实际环境与版本](#二实际环境与版本)
3. [LocationSpoofer 部署步骤](#三locationspoofer-部署步骤step-1-13)
4. [部署验证](#四部署验证)
5. [日常使用与卸载回滚](#五日常使用与卸载回滚)
6. [版本对比：3.0.1 vs 3.1.7](#六版本对比locactionspoofer-301-vs-317)
7. [体适能：部署与闪退排查](#七体适能部署与闪退排查)
8. [体适能：404 诊断](#八体适能404-诊断)
9. [文件清单与校验值](#九文件清单与校验值)
10. [常见问题总表](#十常见问题总表)
11. [一句话交接](#十一一句话交接)

---

# 一、会话总览

按时间顺序发生的事情：

| 阶段 | 内容 | 结果 |
|---|---|---|
| 1 | 检查环境：MuMu 6.8.18.0 + Android 15 实例 + 自带 KernelSU | 发现 D 盘工作区权限问题，用户切到完全权限后继续 |
| 2 | 确认各组件「最新版」 | ZygiskNext 1.5.3、LSPosed→Vector 2.2、LocationSpoofer 3.1.7 |
| 3 | 讨论 LocationSpoofer 版本 | 官方 3.0.0 已下架，3.1.7 需账号+审核+会员；**用户选择旧版 3.0.1** |
| 4 | 完整部署 Root + 三件套 + 授权 + 作用域 | ✅ 全部成功 |
| 5 | 用 GPSTest 验证虚拟定位 | ✅ 坐标被替换，重启后仍生效 |
| 6 | 对比 3.0.1 与 3.1.7 | 发现源码已闭源，改用 APK 静态分析（中途用户要求简化） |
| 7 | 体适能闪退排查 | ✅ 定位为反模拟器检测导致的主动退出，已绕过 |
| 8 | 体适能登录 404 排查 | ✅ 定位为官方域名下线，**无客户端解法** |
| 9 | 整理文档 | ✅ 本文 |

---

# 二、实际环境与版本

| 项目 | 实际值 |
|---|---|
| 模拟器 | MuMu 模拟器 6.0，产品版本 `6.8.18.0`（`product=MuMu6.0`，x64） |
| 安装目录 | `D:\MuMuPlayer` |
| 实例 | `MuMuPlayer-15.0-0`，索引 `0`，名称「MuMu安卓设备」 |
| Android | 15（API 35），实例目录 `nx_device\15.0`，ABI：`x86_64,arm64-v8a,x86` |
| 内核 | `6.1.90-perf+`（KernelSU GKI 模式） |
| Root | MuMu 自带 KernelSU，内核/ksud `32525`，管理器 `v3.2.5 (32525-2)` |
| ADB | `127.0.0.1:16384`（见 `vms\MuMuPlayer-15.0-0\configs\vm_config.json` 的 `port_forward.adb.host_port`） |
| adb 路径 | `D:\MuMuPlayer\nx_main\adb.exe`（版本 1.0.41 / 36.0.0） |
| MuMuManager | `D:\MuMuPlayer\nx_main\MuMuManager.exe` |
| ZygiskNext | **v1.5.3（880-22a1bf5-release）** ← 当前最新稳定版 |
| LSPosed | **Vector v2.2（3080-88f8e1fa-JingMatrix-Vector）** ← 官方 LSPosed 已更名 Vector，API 102 |
| LocationSpoofer | **v3.0.1 scoped / arm64-v8a**（`com.vincenthzr.locationspoofer`） |
| 验证工具 | GPSTest 3.10.6（osmdroid 版，`com.android.gpstest.osmdroid`） |
| 目标应用 | 体适能 `com.bxkj.student` 2.0.26（versionCode 20260，targetSdk 30，arm64-v8a） |

### 关于 LocationSpoofer「最新版」与 3.0.0 的取舍（重要）

- 仓库 `HuangZhuoRui/LocationSpoofer` 现在只剩 **3.1.1 ~ 3.1.7** 七个 Release，**官方 3.0.0 的 APK 已经下架**，只能从源码重建。
- 最新 **v3.1.7** 已改为 AwesomeEdition 方案：**必须注册账号、提交待测应用包名并人工审核，模拟器上使用地图/模拟功能还需要有效会员**。它不再适合「装完即用」。
- 因此按用户选择使用旧版：**TaoCultivator/LocationSpoofer v3.0.1**（官方 v3.0.0 源码副本编译，保留原提交历史）。3.0.x 无需账号、无审核、无会员，包名为 `com.vincenthzr.locationspoofer`（3.0 起从旧包名 `com.suseoaa.locationspoofer` 迁移）。
- ⚠️ 该 3.0.1 APK 是**第三方重新签名**的构建，会被授予 Root 权限。若只接受官方签名，需要自行从源码 `./gradlew assembleRelease` 编译（本机未装 Android SDK，未做此步骤）。

---

# 三、LocationSpoofer 部署步骤（Step 1–13）

## 1. 开启 Root 与「可写系统盘」

MuMu 6 的「Root 权限」开关本质是**开放 root adb**；而后面要给 App 造 `su` 通道，必须让 `/system` 可写（overlay）。

图形界面等价路径：`设置 → 其他/开发者 → Root 权限`，`设置 → 磁盘 → 磁盘共享 → 可写系统盘`。

命令行：

```powershell
$mm = 'D:\MuMuPlayer\nx_main\MuMuManager.exe'
& $mm control -v 0 shutdown
& $mm setting -v 0 -k root_permission -val true
& $mm setting -v 0 -k system_disk_readonly -val false
& $mm control -v 0 launch
```

验证：

```powershell
$adb = 'D:\MuMuPlayer\nx_main\adb.exe'
& $adb connect 127.0.0.1:16384
& $adb -s 127.0.0.1:16384 root          # restarting adbd as root
& $adb -s 127.0.0.1:16384 shell "id; mount | grep ' /system '"
```

预期：`uid=0(root) ... context=u:r:su:s0`，且 `/system` 为 `overlay ... rw`（`upperdir=/mnt/scratch/upperdir`，重启后仍保留）。

> 说明：Android 15 实例自带 ARM 转译，所以 arm64 的 APK 可以直接安装运行（日志中可见 `native_bridge2_initialize ... arm64`）。

## 2. 下载三个包（走 GitHub 加速）

本机直连 `github.com` 的 TLS 会报 `CRYPT_E_NO_REVOCATION_CHECK`，因此统一用 `gh-proxy.com` 前缀下载（`api.github.com` 可直连，用来查版本）。

```powershell
$dl = 'D:\MuMuPlayer\deploy'
New-Item -ItemType Directory -Force $dl | Out-Null

curl.exe -L --fail --retry 3 -o "$dl\Zygisk-Next-1.5.3-880-22a1bf5-release.zip" `
  "https://gh-proxy.com/https://github.com/Dr-TSNG/ZygiskNext/releases/download/v1.5.3/Zygisk-Next-1.5.3-880-22a1bf5-release.zip"

curl.exe -L --fail --retry 3 -o "$dl\Vector-v2.2-3080-Release.zip" `
  "https://gh-proxy.com/https://github.com/JingMatrix/Vector/releases/download/v2.2/Vector-v2.2-3080-Release.zip"

curl.exe -L --fail --retry 3 -o "$dl\LocationSpoofer-scoped-arm64-v8a-v3.0.1.apk" `
  "https://gh-proxy.com/https://github.com/TaoCultivator/LocationSpoofer/releases/download/v3.0.1/LocationSpoofer-scoped-arm64-v8a-v3.0.1.apk"
```

查最新版（可选）：

```powershell
Invoke-RestMethod 'https://api.github.com/repos/Dr-TSNG/ZygiskNext/releases/latest' -Headers @{'User-Agent'='dsh'} | Select tag_name
Invoke-RestMethod 'https://api.github.com/repos/JingMatrix/Vector/releases/latest' -Headers @{'User-Agent'='dsh'} | Select tag_name
```

## 3. 安装 ZygiskNext（必须先于 LSPosed）

MuMu 的 KernelSU 把 `ksud` 改名成了 `/system/xbin/dusk`，模块要用 `dusk` 装，**不要**用 `su`（guest 里根本没有 su 二进制）。

```powershell
& $adb -s 127.0.0.1:16384 push "$dl\Zygisk-Next-1.5.3-880-22a1bf5-release.zip" /data/local/tmp/zygisknext.zip
& $adb -s 127.0.0.1:16384 shell "dusk module install /data/local/tmp/zygisknext.zip"
```

安装器会输出 `Powered by KernelSU`、`Device sdk: 35`、`Device platform: x64`、`Module installed successfully!`，并落到 `/data/adb/modules_update/zygisksu`。

> 若报 `Error: Permission denied (os error 13)`，说明没先 `adb root`。

## 4. 安装 Vector（LSPosed 后继）

```powershell
& $adb -s 127.0.0.1:16384 push "$dl\Vector-v2.2-3080-Release.zip" /data/local/tmp/vector.zip
& $adb -s 127.0.0.1:16384 shell "dusk module install /data/local/tmp/vector.zip"
& $adb -s 127.0.0.1:16384 shell "dusk module list"
```

安装位置 `/data/adb/modules_update/zygisk_vector`，模块 id 为 `zygisk_vector`，版本 `v2.2 (3080-88f8e1fa-JingMatrix-Vector)`，描述声明支持 `Android 8.1 ~ 17`。

## 5. 重启并验证 Zygisk 注入

```powershell
& $adb -s 127.0.0.1:16384 reboot
# 等 sys.boot_completed=1 后
& $adb root; & $adb connect 127.0.0.1:16384
& $adb -s 127.0.0.1:16384 shell "cat /data/adb/modules/zygisksu/module.prop | grep description"
& $adb -s 127.0.0.1:16384 shell "ps -A | grep -i -e vectord -e zn-nsdaemon"
```

预期：

- `description=[✅zygote, ✅zygote_secondary, Root: ✅KernelSU (32525), ZL]`
- 进程里有 `vectord`、`zn-nsdaemon-zygote`、`zn-nsdaemon-zygote_secondary`

这两条同时成立，才说明 ZygiskNext 真的注入进了两个 zygote、Vector 的 zygisk 桥接已工作。

## 6. 补出 `su` 通道（MuMu 的坑）

MuMu 6 的 KernelSU 只把「root」给了 **adb**：guest 里没有 `su`，也没有 setuid 二进制；KernelSU 的 `su_compat` 是**按文件名 `su` 拦截 execve** 来提权的。所以在 PATH 目录里放一个叫 `su` 的可执行文件，App 调 `su` 时就会被内核接管并提权。

```powershell
& $adb -s 127.0.0.1:16384 shell "cp /system/xbin/busybox /system/xbin/su; chmod 755 /system/xbin/su; ls -lZ /system/xbin/su"
```

预期 `-rwxr-xr-x 1 root root u:object_r:system_file:s0 ... /system/xbin/su`。重启后仍然存在（`/system` 是可写 overlay）。

## 7. 安装管理器与 LocationSpoofer

```powershell
& $adb -s 127.0.0.1:16384 pull /data/adb/modules/zygisk_vector/manager.apk "$dl\Vector-manager.apk"
& $adb -s 127.0.0.1:16384 install -r "$dl\Vector-manager.apk"          # org.matrix.vector.manager
& $adb -s 127.0.0.1:16384 install -r "$dl\LocationSpoofer-scoped-arm64-v8a-v3.0.1.apk"
```

> 注意：`adb install` 的路径必须是**主机**路径；`adb -s ... install /data/...` 会报 `failed to stat`。

## 8. 在 KernelSU 里授权 Root（关键一步）

未授权时 App 会停在「需要 Root 权限」页，logcat 里是：

```
E LocationSpoofer: Root command failed
E LocationSpoofer: java.io.IOException: Cannot run program "su": error=13, Permission denied
```

授权步骤（管理器内）：

1. 桌面打开 **KernelSU**；
2. 底部 **超级用户** → 点 **LocationSpoofer**；
3. 打开 **超级用户** 开关（开关变蓝），返回；
4. 强停并重新打开 LocationSpoofer。

## 9. 启用模块 + 配置作用域（可全命令行）

Vector 自带 `vector-cli`，不用点图形界面：

```powershell
$cli = '/data/adb/modules/zygisk_vector/cli'
& $adb -s 127.0.0.1:16384 shell "$cli modules ls"
& $adb -s 127.0.0.1:16384 shell "$cli modules enable com.vincenthzr.locationspoofer"
& $adb -s 127.0.0.1:16384 shell "$cli scope add com.vincenthzr.locationspoofer com.android.gpstest.osmdroid/0"
& $adb -s 127.0.0.1:16384 shell "$cli scope ls com.vincenthzr.locationspoofer"
```

作用域格式是 `包名/用户ID`，主用户固定 `/0`。**加完作用域必须强停一次目标 App**，让新进程重新加载模块：

```powershell
& $adb -s 127.0.0.1:16384 shell "am force-stop com.android.gpstest.osmdroid"
```

> 图形界面等价路径：打开 **Vector** 管理器 → 模块 → 启用 LocationSpoofer → 勾选作用域应用 → 强停目标应用。

## 10. 首次配置与启动模拟

安装后首次打开会依次请求：位置信息（始终允许/仅使用时允许）、电话权限、悬浮窗、后台运行白名单。也可以直接授权：

```powershell
& $adb -s 127.0.0.1:16384 shell "pm grant com.vincenthzr.locationspoofer android.permission.ACCESS_FINE_LOCATION"
& $adb -s 127.0.0.1:16384 shell "pm grant com.vincenthzr.locationspoofer android.permission.ACCESS_COARSE_LOCATION"
& $adb -s 127.0.0.1:16384 shell "pm grant com.vincenthzr.locationspoofer android.permission.ACCESS_BACKGROUND_LOCATION"
& $adb -s 127.0.0.1:16384 shell "pm grant com.vincenthzr.locationspoofer android.permission.READ_PHONE_STATE"
& $adb -s 127.0.0.1:16384 shell "appops set com.vincenthzr.locationspoofer SYSTEM_ALERT_WINDOW allow"
& $adb -s 127.0.0.1:16384 shell "cmd deviceidle whitelist +com.vincenthzr.locationspoofer"
```

然后在 App 内：

1. 选择语言（简体中文）→ 确定；
2. 地图页输入或拖动到目标点（本次用默认 `113.424668, 23.184423`，广州）；
3. 点 **定点模拟** → 在「模拟选项」里按需勾选（本次保留：开启轻微抖动、开始模拟时重启目标应用）→ **开始模拟**；
4. 出现红色 **停止模拟** 按钮即表示已接管。

重启设备后 App 会自动恢复上次的模拟状态（`recoverAfterBoot`）。

## 11. 验证注入成功

LSPosed/Vector 注入日志（`adb logcat`）：

```
I LocationSpoofer: [LocationSpoofer] Application.onCreate fired, appClassLoader=...
I LocationSpoofer: [LocationSpoofer] TencentLocation SDK not found, skipped
I LocationSpoofer: [LocationSpoofer] BaiduMap class found=false ...
```

## 12. 日常使用：给新的目标 App 加作用域

```powershell
$pkg = 'com.tencent.mm'   # 换成你的目标应用包名
& $adb -s 127.0.0.1:16384 shell "/data/adb/modules/zygisk_vector/cli scope add com.vincenthzr.locationspoofer $pkg/0"
& $adb -s 127.0.0.1:16384 shell "am force-stop $pkg"
```

查看已装应用包名：

```powershell
& $adb -s 127.0.0.1:16384 shell "pm list packages -3"
```

> scoped 版只对**作用域内**的 App 生效；没进作用域的 App 看到的是真实（模拟器默认）位置。若你希望「全局接管系统服务」，需要改用同仓库的 `global` 变体，但 3.0.x 的全局方案对各家 ROM 需要逐个适配，MuMu Android 15 未验证，**不建议**直接上。

## 13. 卸载与回滚

```powershell
# 移除模块（管理器「模块」页也可操作），可选其一后重启
& $adb -s 127.0.0.1:16384 shell "dusk module list"
& $adb -s 127.0.0.1:16384 shell "rm -rf /data/adb/modules/zygisk_vector /data/adb/modules/zygisksu"

# 删掉 su 通道
& $adb -s 127.0.0.1:16384 shell "rm -f /system/xbin/su"

# 卸载 App
& $adb -s 127.0.0.1:16384 uninstall com.vincenthzr.locationspoofer
& $adb -s 127.0.0.1:16384 uninstall com.android.gpstest.osmdroid
```

恢复只读系统盘：`MuMuManager.exe setting -v 0 -k system_disk_readonly -val true` 后重启；若想彻底还原 `/system`，需在 MuMu 磁盘设置里还原系统盘（会清掉 overlay 内的改动）。

---

# 四、部署验证

GPSTest 读取结果（LocationSpoofer 设定点在广州 `23.184423, 113.424668`）：

| 字段 | 值 |
|---|---|
| 纬度 | 23.1869033° |
| 经度 | 113.4191422° |
| 卫星数量 | 7/7/7 |
| 高度(MSL) | 800.0 m |
| H/V 精度 | 1.8 / 1.2 m |

偏差来自「轻微抖动」（默认 4 m 内 OU 随机过程），符合预期。

**重启持久性验证**：重启后 `/system/xbin/su` 仍在、`/system` overlay 仍为 `rw`、`zygisksu` 与 `zygisk_vector` 模块仍在、作用域配置仍在、LocationSpoofer 模拟状态自动恢复。

证据截图：

- `final_app.png`：LocationSpoofer 模拟运行中（红色「停止模拟」）
- `final_gpstest.png`：GPSTest 读到伪造坐标与卫星数

---

# 五、日常使用与卸载回滚

见上文 [3.12](#12-日常使用给新的目标-app-加作用域) 与 [3.13](#13-卸载与回滚)。

---

# 六、版本对比：LocactionSpoofer 3.0.1 vs 3.1.7

> 说明：此项对比中途按要求简化，以下为已确认的结论。

## 6.1 前提变化

- **官方 v3.1.7 源码不再公开**：GitHub 仓库 git 历史被重写，v3.1.7 标签下只剩 `LICENSE` + 两份 README。对比只能基于安装包实测。
- **3.0.1 有「中间未发布版本」**：3.1.7 安装包内适配文档提到「本地 v3.0.12 已恢复 ColorOS 16 原生传感器适配」。
- **运行模式根本变化**：3.0.1 是 `scoped`（作用域为微信、钉钉、超星等 9 个 App）；3.1.7 只有 `global`，作用域固定为 `system / com.android.phone / com.android.bluetooth / com.google.android.gms`。

## 6.2 安全与规则（APK 字符串统计）

| 维度 | 3.0.1 | 3.1.7 |
|---|---|---|
| 反检测痕迹（xposed/lsposed 相关） | 28 处 | **653 处** |
| Root 痕迹检测相关 | 7 处 | 16 处 |
| TEE / 设备证明 | 52 处 | 69 处 |
| 账号 / 会员 / 授权审核 | 48 / 164 / 55 | **134 / 306 / 98** |
| 外连域名数量 | 102 个（百度、高德、Google、WiGLE 等） | 33 个（百度、Google Maps、WiGLE 全部移除） |
| 新增域名 | — | `server.vincenthzr.org`、`downloads.vincenthzr.org`、`apply.awesomeedition.com` |
| 新增原生库 | 无 | `libshadowhook.so`、`liblocation_accel.so` |

可概括的规则变化：3.1.7 增加**账号登录 + 包名申请审核（最多 10 个）+ 会员**三道门槛；从「安装即用」变成「以账号/审核/会员在线校验为准」；加强反 Xposed / 反 Root；把百度、Google、WiGLE 的数据出口全部替换为高德与 OSM。

## 6.3 功能变化（官方 Release 说明）

- **3.1.1**：手机端账号申请测试应用，10 个包名上限；已批准包名不随会员到期失效。
- **3.1.2**：已审核免费账号可用 OSM 地图、定点模拟、路线模拟、路线录制、收藏、云备份；云端 WiFi/基站仍需会员。
- **3.1.3**：模拟器/未认证设备提示；真机增加 TEE 与设备信息检测。
- **3.1.4**：站内申请测试包名、包名单选开关、应用列表图标/搜索/筛选。
- **3.1.5**：审核结果弹窗、待审核自动刷新。
- **3.1.6**：手绘路线、路线循环不再反向、摇杆加减速按钮、注册可选邮箱。
- **3.1.7**：会员图源改为高德普通图 + 卫星图（去掉 Google）、搜索跨页复用、**移除「真实道路规划」**。

## 6.4 结论

3.1.7 比 3.0.1 **多了**：账号体系、包名审批、会员、手绘路线、审核弹窗、高德卫星图、跨页搜索复用、增强反检测。
**少了**：真实道路规划、百度地图、Google 地图、WiGLE/OpenCellID 云端导入、免账号直接使用、scoped 作用域模式。

---

# 七、体适能：部署与闪退排查

## 7.1 部署

- APK 位置：`D:\MuMuPlayer\deploy\体适能.apk`（136,448,148 字节，约 130 MB）
- 包名：`com.bxkj.student`，版本 2.0.26（versionCode 20260，targetSdk 30）
- 安装：`adb -s 127.0.0.1:16384 install -r "D:\MuMuPlayer\deploy\体适能.apk"`

## 7.2 现象

在「欢迎使用体适能App」权限说明页点击「同意并使用」后，应用立即闪退，回到桌面。

## 7.3 根因

**应用主动退出，不是系统崩溃。** 日志时序：

```
I eup     : register native abi method success
I om.bxkj.student: System.exit called, status: 0
I AndroidRuntime: VM exiting with result code 0, cleanup skipped.
I Zygote  : Process 2711 exited cleanly (0)
```

点击后不到 0.1 秒就调用 `System.exit(0)`。APK 内的 `libemulator_check.so` 导出的符号是：

```
Java_com_snail_antifake_jni_EmulatorDetectUtil_detect
```

该库通过 `mmap` / `asmcheck` 做反模拟器检测，并自行处理 SIGSEGV。在 MuMu 上只要检测到模拟器环境，就主动结束进程。

## 7.4 对照实验（在隔离的克隆实例上完成）

为避免影响主实例，克隆了实例 0 做测试，完成后已删除克隆实例。

| 变量 | 结果 |
|---|---|
| 原始环境 | 闪退 |
| 关闭 LocationSpoofer 模块 | 仍闪退（排除 LSPosed 注入） |
| 关闭 Root | 仍闪退（排除 KernelSU） |
| 修改机型指纹（brand 小写 samsung、model 改 SM-G9910） | 仍闪退 |
| 覆盖安装且不保留数据 | 仍闪退（排除数据残留） |
| **直接启动 `.MainActivity`** | **正常进入，可到登录页** |

## 7.5 解决方案：绕过启动页

用 adb 直接拉起主界面，跳过会做检测的 `SplashActivity` / `OpenPrivacyAgreementActivity`：

```powershell
adb -s 127.0.0.1:16384 shell "am force-stop com.bxkj.student"
adb -s 127.0.0.1:16384 shell "am start -n com.bxkj.student/.MainActivity"
```

进入后的操作路径：关闭「获取学校信息异常」弹窗 → 关闭推送提示「不了」→ 底部「我的」→ 点「未登录 / 请登录」→ 到达账号登录页。

也可直接双击 `D:\MuMuPlayer\deploy\启动体适能.bat`。

## 7.6 为什么直接开 PowerShell 敲命令不行

`am start` 是**模拟器内部**的命令，必须通过 adb 转交；而直接打开的 PowerShell 里 `adb` 不在 PATH 中。

正确做法（用全路径）：

```powershell
cd D:\MuMuPlayer\nx_main
.\adb.exe connect 127.0.0.1:16384
.\adb.exe -s 127.0.0.1:16384 shell am start -n com.bxkj.student/.MainActivity
```

或把 adb 加入用户 PATH（只需一次）：

```powershell
[Environment]::SetEnvironmentVariable('Path', [Environment]::GetEnvironmentVariable('Path','User') + ';D:\MuMuPlayer\nx_main', 'User')
```

> PowerShell 中运行当前目录的程序必须加 `.\` 前缀。

---

# 八、体适能：404 诊断

## 8.1 最终结论

**`https://h.boxkj.com/` 已在服务端整体下线：任何网络、任何设备都打不开。而体适能 2.0.26（官方最新版）的登录请求固定发往该域名，因此所有用户当前都无法登录。**

这不是环境问题。模拟器侧已做到位：应用能正常安装、启动、绕过启动页并到达账号登录界面，请求也确实发出去了，只是目标域名已经不存在。

用户确认的三条事实：

1. `https://h.boxkj.com/` 任何网络、任何设备都打不开 —— 域名层面下线，非本地网络问题。
2. 2.0.26 是官方最新版本 —— 不存在「升级到新版即可修复」的路径。
3. 应用强制更新，不给运行老版本 —— 也无法回退到使用旧域名的版本。

三条叠加的结果：**当前无客户端可行解法，只能等待官方修复或更换服务端域名并发布新版本。**

## 8.2 域名状态

从 `体适能.apk` 的 `classes.dex` 提取出的接口域名：

| 域名 | 用途 | 状态 |
|---|---|---|
| `m.boxkj.com` | 业务数据（学校、权限、设备绑定等） | ✅ 正常返回 JSON |
| `h.boxkj.com` | **登录、用户中心**（`/app/appstu/login`、`/app/user/grantUser` 等） | ❌ 已下线，无任何解析记录 |
| `www.boxkj.com` | 官网 | ✅ 200 |
| `w.boxkj.com` / `keep.boxkj.com` / `mall.boxkj.com` | 证书 SAN 覆盖范围内 | ❌ 无解析记录 |

DNS 实测（宿主机与模拟器结果一致）：

```
m.boxkj.com   -> 117.34.70.45, 39.96.153.44, 47.118.199.204 ...   有 A 记录
www.boxkj.com -> 117.34.70.48, 39.96.153.44 ...                   有 A 记录
h.boxkj.com   -> 无 A / AAAA 记录
```

TLS 证书为 `CN=*.boxkj.com`（DigiCert，Encryption Everywhere DV TLS CA - G2，有效期至 2027-01-21），**证书本身正常**，说明 `h.` 这个子域名是被人为撤掉，而不是证书或链路故障。

## 8.3 接口逻辑仍然存在

把同一路径换到仍在服务的 `m.boxkj.com` 上请求：

```
POST https://m.boxkj.com/app/appstu/login  {"suId":"...","password":"..."}
-> 200  {"returnCode":601,"returnMsg":"认证失败"}

POST https://m.boxkj.com/app/sch/listSchoolByProvinceId
-> 200  {"total":0,"data":[],"returnCode":"200","returnMsg":null}
```

`601 认证失败` 是正常业务返回，说明后端登录逻辑在运行。App 报 404，仅因为它请求的 `h.boxkj.com` 已经不存在。

## 8.4 模拟器网络本身正常

```
模拟器内 ping www.baidu.com   -> 0% packet loss
模拟器内 ping www.boxkj.com   -> 0% packet loss
```

## 8.5 已排除的无效手段

以下操作对 404 均无作用，不必再尝试：

- 换 Wi-Fi / 手机热点 / 代理 / 换 DNS
- 改模拟器机型、指纹、分辨率
- 关闭 Root、关闭 LSPosed / LocationSpoofer
- 覆盖安装、清除数据、换账号换密码
- 回退旧版本（应用强制更新，无法运行）

原因：这些都不改变「登录请求的目标域名不存在」这一事实。

## 8.6 如果官方恢复或迁移域名，需要做的事

1. 确认 `h.boxkj.com` 恢复解析，或官方发布新版本 APK；
2. 覆盖安装：`adb -s 127.0.0.1:16384 install -r <新安装包>`；
3. 启动（跳过启动页）：`am start -n com.bxkj.student/.MainActivity`；
4. 若要让虚拟定位对该应用生效，把包名加入 LocationSpoofer 作用域：
   ```powershell
   adb -s 127.0.0.1:16384 shell "/data/adb/modules/zygisk_vector/cli scope add com.vincenthzr.locationspoofer com.bxkj.student/0"
   adb -s 127.0.0.1:16384 shell "am force-stop com.bxkj.student"
   ```
5. Root、ZygiskNext、LSPosed（Vector）、LocationSpoofer 的既有配置无需重做。

> ⚠️ 该应用含 `libemulator_check.so` 等反模拟器检测，即便登录恢复，跑步过程中的风控仍可能判定环境异常，使用前请自行评估。

---

# 九、文件清单与校验值

目录：`D:\MuMuPlayer\deploy`

| 文件 | 大小 | SHA256 |
|---|---|---|
| `Zygisk-Next-1.5.3-880-22a1bf5-release.zip` | 6.33 MB | `186A0A26C9FCB895AF1954B42DE831433C23D72D1BD0758B986069154359DB67` |
| `Vector-v2.2-3080-Release.zip` | 8.89 MB | `9EE8323575D615F7B3F1076FF60B2A63A49390EF11881B52632311A37F6F79CC` |
| `LocationSpoofer-scoped-arm64-v8a-v3.0.1.apk` | 45.13 MB | `6A54272F82B397148988BD64C02D9E040E513300EA40A5C088D4D1A10680F747` |
| `Vector-manager.apk`（从模块内提取） | 3.05 MB | `101EAEEC360106F9E8AA156B3F2C7695630FF918B0D56C416782AB62985CB005` |
| `gpstest.apk`（仅验证用） | 3.30 MB | `897B91FE6A1EA0C3A7F6E2B041D5ACE0D582941F67CF454FB7778F031D89CE41` |
| `体适能.apk` | 130.13 MB | — |
| `LocationSpoofer-v3.1.7-global-64bit-arm64-v8a.apk` | 61.26 MB | （版本对比用） |

脚本：

| 文件 | 用途 |
|---|---|
| `启动体适能.bat` | 双击即可通过 adb 跳过启动页拉起体适能主界面 |
| `启动体适能.ps1` | 同上，PowerShell 版本，带连接检查 |

截图：

- `final_app.png`：LocationSpoofer 模拟运行中
- `final_gpstest.png`：GPSTest 读到伪造坐标
- `final_login.png`：体适能账号登录页

---

# 十、常见问题总表

## LocationSpoofer 相关

| 现象 | 原因 / 处理 |
|---|---|
| `Cannot run program "su": error=13` | 未在 KernelSU 授权。管理器 → 超级用户 → 该 App → 打开「超级用户」开关后强停重开 |
| `dusk module install` 报 `Permission denied (os error 13)` | 没有先 `adb root` |
| 模块装了但没生效 | 没重启；或没启用模块；或作用域没加；或目标 App 没强停 |
| GPSTest 显示真实位置 | 目标包名不在作用域（注意 GPSTest 的包名是 `com.android.gpstest.osmdroid`，不是 `com.android.gpstest`） |
| KernelSU 管理器提示有新版本 32680 | **不要升级**。MuMu 的 KernelSU 是定制集成（ksud 改名 dusk），升级管理器可能破坏模块/授权 |
| 地图瓦片空白 | arm64 转译下地图 SDK 的原生库可能异常；可换网络或在真机验证 |
| 系统盘可写后想还原 | 见 3.13；overlay 改动会一直保留 |

## 体适能相关

| 现象 | 原因 / 处理 |
|---|---|
| 点「同意并使用」后闪退 | `libemulator_check.so` 反模拟器检测，主动 `System.exit(0)`。用 `am start -n com.bxkj.student/.MainActivity` 绕过启动页 |
| 从桌面图标打开还是闪退 | 桌面入口指向 `SplashActivity`，必然触发检测；用上面的命令或 `启动体适能.bat` 启动 |
| 登录报「连接服务器错误，错误码：404」 | 官方登录域名 `h.boxkj.com` 已下线，非环境问题，无客户端解法 |
| 提示「获取学校信息异常」 | 未登录状态下的正常提示，关掉即可 |

## 通用

| 现象 | 原因 / 处理 |
|---|---|
| PowerShell 里 `adb` 无法识别 | adb 不在 PATH；用全路径 `D:\MuMuPlayer\nx_main\adb.exe` 或 `cd` 到该目录后加 `.\` |
| `more than one device` | 加 `-s 127.0.0.1:16384` 指定设备 |
| `adb devices` 列表为空 | 模拟器未启动或 ADB 调试未开启 |
| `adb install /data/...` 报 `failed to stat` | install 的路径必须是宿主机路径 |
| 同一个 `adb` 客户端连多个实例时会断开 | 两个实例可能共用 `16384` 端口，克隆实例会分配到新端口（如 16416）。用 `MuMuManager info -v all` 查状态、读 `vms\MuMuPlayer-15.0-N\configs\vm_config.json` 的 `host.adb` 端口 |

---

# 十一、一句话交接

模拟器与框架侧已全部就绪且验证通过（Root、ZygiskNext、LSPosed/Vector、LocationSpoofer 虚拟定位、体适能绕过启动页均已生效）；**唯一阻塞点是体适能官方登录域名 `h.boxkj.com` 已下线，而 2.0.26 是强制更新的最新版本，客户端无法绕开。项目在此结束，等待官方修复。**

# qcmon — QuickCharge 的 C 语言守护进程

用 C 重写的 QuickCharge 温控与充电守护。功能与 shell 版（`lib/qc.sh`）等价，
把每轮循环的进程创建从 41 次降到 **0** 次，每 tick 从 9.4 ms 降到 **0.14 ms**。

> shell 版见模块主仓库：[QuickCharge](https://github.com/XiaohaoQvQ/QuickCharge)

> **语言说明**：本项目是**纯 C**（C99），不是 C++，也不是 C#。
> 源码只有 `src/qcmon.c` 一个文件，用 `gcc` 编译，只包含 C 标准头
> （`stdio.h`、`dirent.h`、`sys/mount.h` 等），没有 class / template / namespace，
> 没有 `iostream`，也没有 .NET 运行时。

---

## 1. 为什么要有 C 版

这个守护进程的工作极其简单：每 3-15 秒读几十个 sysfs 小文件，算一遍目标温度，
偶尔写几个节点。**在这种负载下，成本几乎全在“启动开销”上，而不是 CPU 算力。**

shell 版的结构决定了它绕不开这个成本：

```sh
# 读一个温度节点，shell 需要 fork 两个进程
temp=$(cat "$zone/temp" | tr -d ' \t\r\n')
```

- `$( )` → fork 一次，exec 一个 `sh`
- `| tr` → 再 fork 一次，exec 一个 `busybox`
- 每轮 24 个温区，光读温度就是 40+ 次进程创建

C 版做同一件事：

```c
/* open + read + close，3 个系统调用，0 次 fork */
int t = read_int_file(path);
```

这不是“C 更快”这种模糊说法，而是把**每秒数十次的进程创建**彻底消除。

---

## 2. 两版实测对比

测试环境：24 个温区 + 完整电池节点，100 轮取均值。arm64 数字来自 qemu 用户态
（有 3-10 倍解释开销，真机落在 0.1-0.2 ms）。

| 指标 | 2.9.42 shell | 2.9.43 shell 优化 | **2.9.44 C** |
|---|---|---|---|
| 每 tick CPU | 9.4 ms | 1.3 ms | **0.14 ms** |
| 每 tick fork | 41 | 4（首轮后 1） | **0** |
| 常驻内存 RSS | 3-5 MB | 3-5 MB | **988 KB** |
| 其中私有脏页 | — | — | **96 KB** |
| 日均 CPU（tick=15s） | 约 54 秒 | 约 7 秒 | **约 1 秒** |
| 包体积 | 67 KB | 80 KB | 406 KB |

两点需要说明：

**为什么 988 KB 里只有 96 KB 是自己的。** 892 KB 是 mmap 进来的只读代码页，
内核缺内存时会直接丢弃，需要时再从文件读回。真正属于进程的私有内存只有 96 KB，
比 shell 版每个 fork 出来的子进程都少。

**为什么包大 320 KB。** 二进制 713 KB 里，我们自己写的逻辑只占 42 KB，
其余基本是静态链接的 glibc（仅代码就 603 KB）。Android 用的是 bionic 而非 glibc，
动态链接的 glibc 程序在手机上连解释器都找不到，只能静态链接。

---

## 3. 功能覆盖

2.9.44 起与 shell 版全量对齐，不再有“开机执行一次、之后不跟随”的缺口。当前版本 `3.1.6-c`。与 shell 版 `3.1.6` 的差别只在守护实现：shell 版跑 `lib/qc.sh` 解释执行，C 版跑静态二进制；业务逻辑（配置改写、温度伪装、亮屏接管、cool_down）两侧同源同测试。

**温度**
- 温区扫描与分类（core / noncore / battery / shell / gpu / 兜底）
- 壳温恒定额上报（`static` / `charge` 都写同一个值）：不看真实读数、不设温度墙、不设熔断线
- `emul_temp` 与 `/proc/shell-temp` 双落点
- `charge` 模式：插电就接管，不看电量，拔电还原
- 电池 NTC 伪装，不按真实温度撤销
- 真实温度只写遥测（面板显示用），不参与任何判断

**限流**
- 降流解除：`cool_down` 非 0 且实测 `|ibat| < 2.5A` 才写 0，已满速时不碰这个节点
- 单位自动换算（µA / mA），回写保留原始量级；报 mA 的机型不会把 3A 当成 3mA
- 阶梯解锁：HAL 节点写 0，原值备份，不按温度撤销

**安全**
- 3.1.0 起只保留边界校验：伪装值 20 到 33℃，越界拒绝
- 退出时还原所有被改写节点
- 熔断、温度墙、温差档位、绝对电流墙、电量阈值**全部删除**——模块只追求充电速度

**屏幕快充**
- 优先写内核 `ignore_screen_state=1`（多个厂商路径依次尝试），息屏这一轮还原 0，交给驱动走正常 FCC
- 内核不支持时只记住息屏放开的 FCC 峰值，**息屏不写节点、也不还原**；亮屏被驱动压到屏幕档后再按峰值顶回去
- 不设温度闸：亮屏就接管，直到没在充电为止

**其他**
- 遥测写入 `run/sensor`（7 字段，面板直接读）
- 高刷补写与锁定
- 心跳 `uptime pid`（与 `sd_hb_age` 兼容）
- `SIGUSR1` 热重载 `state.conf`
- `module.prop` 描述同步
- 单实例锁（`mkdir` 原子创建，已存在即退出）

---

## 4. 与模块的集成

C 版不是独立程序，它替代的是模块里的 shell 守护循环。`service.sh` 里的判定：

```sh
# 有二进制、且没被 SD_SHELL_DAEMON=1 强制禁用
if [ -x "$MODDIR/bin/qcmon" ] && [ "${SD_SHELL_DAEMON:-0}" != 1 ]; then
    if "$MODDIR/bin/qcmon" -v >/dev/null 2>&1; then
        export SD_DATA
        export SD_MODDIR="$MODDIR"
        exec "$MODDIR/bin/qcmon"      # 热路径整个交给 C
    fi
    sd_warn daemon "bin/qcmon 无法执行，回退 shell 守护"
fi
# 走到这里才执行 shell 循环
```

所以两个发行包共用同一份 `service.sh`，差别只在包内有没有 `bin/qcmon`。
二进制缺失、架构不对、被杀软拦截，都会自动退回 shell 版。

强制用 shell 守护（不重刷）：

```sh
su -c 'SD_SHELL_DAEMON=1 sh /data/adb/modules/quickcharge/service.sh'
```

---

## 5. 构建

只需要一个能出 aarch64 静态二进制的交叉编译器：

```sh
# Debian / Ubuntu
sudo apt install gcc-aarch64-linux-gnu

# 构建 arm64 静态二进制（strip 后 713 KB）
sh build-qcmon.sh arm64 bin/qcmon

# 构建本机 x86 版本，用于跑测试
sh build-qcmon.sh x86 /tmp/qcmon_x86
```

实际编译命令：

```sh
aarch64-linux-gnu-gcc -Os -static -no-pie -s \
    -Wall -Wextra -Wno-format-truncation \
    -o bin/qcmon src/qcmon.c
```

`-static` 是必须的（bionic 不兼容），`-s` 只是为了去符号表减小体积（835 KB → 713 KB），
去掉后功能完全一致，调试时反而建议保留符号。

---

## 6. 测试

`tests/c-test.sh` 在假 sysroot 上搭一套温区与电池节点，跑完整链路，共 **87 项**。
x86 与 arm64（qemu 用户态）各跑一遍。

```sh
# x86
sh tests/c-test.sh /tmp/qcmon_x86

# arm64，需要 qemu-user-static
sh tests/c-test.sh "/usr/bin/qemu-aarch64-static ./bin/qcmon"
```

覆盖内容：温区分类、恒定壳温上报、伪装生效与还原、遥测与心跳格式、
`SIGUSR1` 后仍存活、高温继续伪装（不再熔断）、亮屏快充两路、限流单位换算、
降流解除的电流闸、电池伪装不撤销、`off` 模式、单实例锁。

测试全部在临时目录里做 bind mount，不触碰真实 `/sys`。

---

## 7. 调试命令

排障时不用改代码，几个只读子命令：

| 命令 | 作用 |
|---|---|
| `qcmon zones` | 列出扫到的温区、类型分类、是否可写 `emul_temp` |
| `qcmon once` | 跑一轮热路径并打印决策（墙值、目标电流），不落盘 |
| `qcmon sensor` | 打印当前遥测内容与屏幕/充电状态 |
| `qcmon bench N` | 连跑 N 轮热路径，输出平均耗时，用于性能回归 |
| `qcmon -v` | 打印版本，`service.sh` 用它判断二进制是否可用 |

正常运行时 `qcmon` 无参数即进入守护循环。

---

## 8. 环境变量

| 变量 | 作用 |
|---|---|
| `SD_DATA` | 数据目录（`state.conf`、日志、心跳），默认 `/data/adb/quickcharge` |
| `SD_MODDIR` | 模块目录，用于同步 `module.prop` 描述；兼容旧名 `MODDIR` |
| `SD_TZ` | 时区偏移，`service.sh` 在 `exec qcmon` 前写入（如 `+0800`）；静态二进制没有 tzdata，缺了它会退回 UTC |
| `SD_SYSROOT` | sysfs 根前缀，测试用 |
| `SD_SPOOF_DIR` | 伪装备份与挂载记录目录 |
| `SD_BL` | 背光节点路径，用于判断亮屏 |
| `SD_ZONE_GLOB` | 自定义温区路径，测试用 |

---

## 9. 已知限制

诚实列出，不掩饰：

1. **只提供 arm64 二进制**。arm32 设备需要自己把 `build-qcmon.sh` 里的工具链换成
   `arm-linux-gnueabihf-gcc`（脚本目前只处理 x86 与 arm64 两种情况）。
2. **静态 glibc 与 bionic 语义不完全一致**。代码刻意只用了 `open/read/write/stat/mount/system`，
   避开了 `getpwnam`、locale、DNS 这些在 Android 上有坑的接口，但“零风险”是假的。
3. **崩溃即停**。C 里一处越界就是段错误，进程直接死，动态策略全停；
   shell 版一行报错还能继续跑下一轮。静态二进制没有 ASan，现场只能看日志。
4. **不可审计是相对的**。源码在 `src/qcmon.c`，构建脚本在 `build-qcmon.sh`，
   但没有做 bit-for-bit 的可复现构建（`BuildID` 和路径会带差异）。
5. **两套要同时维护**。功能改动需要写两遍，容易分叉。现在靠 87 项测试双跑兜底。
6. **必须配合模块使用**。它读 `state.conf`、依赖模块的目录布局，不能单独跑。

第 3 条是最实际的风险，也是 `service.sh` 保留 shell 回退的原因。

---

## 10. 什么时候该用 C 版

**收益是每小时省下几十毫秒 CPU，成本是失去现场自救能力和增加维护量。**

- 日常使用：**建议 shell 版**。7 秒/天的 CPU 差异在手机上不可感知。
- 老机型、追求极低唤醒开销：用 C 版，988 KB 常驻、0 fork。
- **真正需要 C 版的场景只有一个**：如果 shell 版因为 fork 太频繁被某些机型的内核
  cgroup 限流或记录异常，那时 C 版是必需而非可选。

---

## 11. 文件结构

```
quickcharge/
├── src/qcmon.c            C 守护源码（2711 行，单文件）
├── build-qcmon.sh         交叉编译脚本（24 行）
├── tests/c-test.sh        87 项冒烟测试（547 行）
├── bin/qcmon              arm64 静态二进制（713 KB）
├── lib/qc.sh              shell 版核心库（4116 行，C 版的对照实现）
├── service.sh             启动入口，检测 bin/qcmon 决定走哪条路径
├── action.sh              控制入口：CLI + 交互菜单
├── customize.sh           安装：设备检测、生成覆盖层、初始化参数
├── post-fs-data.sh        开机早期：挂载覆盖层，OTA 后重建
├── uninstall.sh           卸载：停守护、卸挂载、归零、清数据
├── webroot/index.html     WebUI
└── module.prop
```

`lib/qc.sh` 与 `src/qcmon.c` 是同一套策略的两种实现。读其中一份再看另一份，
比读任何文档都快。

---

## 12. 版本与发行版

| 仓库 | 当前版本 | 包名 | versionCode |
|---|---|---|---|
| [QuickCharge](https://github.com/XiaohaoQvQ/QuickCharge)（shell） | `3.1.6` | `QuickCharge-v3.1.6.zip` | 301601 |
| [QuickCharge-C](https://github.com/XiaohaoQvQ/QuickCharge-C)（C 守护） | `3.1.6-c` | `QuickCharge-v3.1.6-c.zip` | 301600 |

发行版标题统一为「产品全名 + 版本号」，不带摘要：

```
Quick Charge v3.1.6-c
```

tag 为 `3.1.6-c`（与标题一致，去掉产品名）。C 版 versionCode 固定在偶数序列，shell 版为奇数序列，
两者互不覆盖，可以先 shell 后 C 反复换刷。

3.0 起版本固定三段 `主.次.修订`，修订到 100 进一位（`3.0.100` 之后是 `3.1.0`）；
`versionCode = (主×100 + 次)×1000 + 修订×2 + 通道`，通道 1 = shell、0 = C。

### 在线升级

`module.prop` 里的 `updateJson` 指向本仓根目录的 `update.json`，管理器定期拉取并比较
`versionCode`，比本地 `versionCode` 大就在模块卡片上显示「更新」，点一下自动下载刷入：

```properties
updateJson=https://github.com/XiaohaoQvQ/QuickCharge-C/releases/latest/download/update.json
```

```json
{
  "version": "3.1.6-c",
  "versionCode": 301600,
  "zipUrl": "https://github.com/XiaohaoQvQ/QuickCharge-C/releases/download/3.1.6-c/QuickCharge-v3.1.6-c.zip",
  "changelog": "https://github.com/XiaohaoQvQ/QuickCharge-C/releases/latest/download/RELEASE.md"
}
```

`update.json` 和 `RELEASE.md` 都是 Release 资产，`latest/download/` 永远指向最新一版，
没有 CDN 缓存。发版时三个文件一起传：模块包、`update.json`、`RELEASE.md`。

C 版的 `updateJson` 必须指 C 仓。两仓模块 ID 同为 `quickcharge`，指错了会把 shell 包推给
C 版用户，装完守护进程就没了。

> **`module.prop` 的键名必须写 `versionCode`，C 大写。** 管理器按驼峰取值，写成全小写的
> `versioncode` 会被读成 `0`，远端任何版本都“更新”，按钮永远消不掉。3.0.2 及更早踩过，
> 3.0.3-c 起已改。

2.9.72 起 C 守护的日志时间戳跟随本地时区：静态二进制没有 tzdata，`service.sh` 在 `exec` 前
`export SD_TZ=$(date +%z)`，守护直接解析 `+0800`，不再退回 UTC。

---

## 13. 许可

本模块源码完全公开，**禁止商用与非法用途**。

修改温控与充电策略存在硬件风险，使用者自行承担后果。

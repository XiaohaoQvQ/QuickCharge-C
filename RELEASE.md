# Quick Charge v2.9.45-c（C 守护版）

> 对应包：`QuickCharge-v2.9.45-c.zip` · `versioncode 29451`

修一个只影响**元模块用户**的显示缺陷：模块功能一直是好的，但模块列表里的描述写着
"未生效"，配置项数也变成了 0。

## 现象

装好 C 版、功能正常，模块列表里却是：

```
QC · 温控档位 10/10 · 高刷关闭 · 未生效
```

熟悉的"配置 49 项"不见了。因为写着"未生效"，很容易误判成模块没工作。

如果你没用元模块（配置文件由模块自己 bind 挂载），不会遇到这个问题——这版的功能
与 2.9.44 完全一致，升级与否都可以。

## 原因

配置项数是靠数清单文件得来的，但两种挂载方式的清单放在不同文件、计数规则也不同：

| 挂载方式 | 清单文件 | 计数规则 |
|---|---|---|
| 模块自己 bind 挂载 | `mounts.list` | 去重后计数（同一路径重复挂载只算一项） |
| 元模块（NoMount / Magic Mount / Mountify / Hybrid Mount） | `nm.list` | 不去重，按行数 |

shell 版一直两条分支都实现，C 版在 2.9.43 移植时只实现了 `mounts.list`。

装了元模块时，配置文件由元模块注入，`mounts.list` 是空的 → C 版数出 0 → 输出"未生效"。

**数出 0 只说明统计没取到，配置本身是好的**，所以功能一直正常。

## 修复

**1. 补上元模块分支**

判断逻辑与 `lib/qc.sh` 的 `sd_nm_active` / `sd_mm_active` 逐条对齐：

- 扫 2 个 NoMount 目录、7 个 Magic Mount / Mountify / Hybrid Mount 目录
- 目录存在、且不带 `disable` / `remove` 标记，才算激活
- 激活且 `nm.list` 非空 → 按行数统计
- 否则回退 `mounts.list` 去重统计
- 两者都为空 → 显示"配置未挂载"

**2. 文案从「未生效」改为「配置未挂载」**

两版一起改。原来的说法容易被理解成"模块整体没生效"，改成"配置未挂载"后指向明确：
**进程在跑，只是这次没挂到配置**。看到这句该去查清单，而不是重装模块。

**3. 新增 `SD_ADB` 环境变量**

默认 `/data/adb`，只为测试用——把元模块目录指到临时根，不然跑测试会去扫真实的
`/data/adb/modules`。正常使用不需要碰它。

## 验证

用假 sysroot 复现了四种场景：

| 场景 | 2.9.44 | 2.9.45 |
|---|---|---|
| 普通挂载，`mounts.list` 49 行 | 配置 49 项 | 配置 49 项 |
| 元模块，`nm.list` 37 行 | 未生效 | **配置 37 项** |
| 元模块带 `disable` 标记 | 未生效 | **回退 `mounts.list`，12 项** |
| 两者都为空 | 未生效 | 配置未挂载 |

冒烟用例从 46 项加到 **49 项**，x86 与 arm64（qemu 用户态）各跑一遍，全部通过。

## 升级

直接覆盖安装，配置、日志、参数全部保留。

- 2.9.44-c → 2.9.45-c：`versioncode` 29441 → 29451，正常升级
- 从 shell 版换过来：29450 → 29451，也是升级

**从 shell 包换成 C 包、或从 C 包换回 shell 包时注意**：部分管理器覆盖安装不清理旧文件。
如果保留着 `bin/qcmon`，`service.sh` 会继续跑 C 版，你以为切回来了其实没有。

```sh
ls -l /data/adb/modules/quickcharge/bin/qcmon
```

存在就删掉再重启。想确认当前跑的是哪一版，看日志里有没有"切换到 C 守护 qcmon"这句：

```sh
su -c 'grep qcmon /data/adb/quickcharge/log.txt | tail -3'
```

## 文件

| 项 | 值 |
|---|---|
| 大小 | 398,342 字节 |
| 文件数 | 19（含 `bin/qcmon`，713 KB 静态 arm64） |
| 架构 | **仅 arm64**（老 arm32 设备请用 shell 版） |
| 运行时内存 | 常驻约 988 KB，其中私有内存约 96 KB |
| 每轮开销 | 0.14 ms / 0 次 fork |

## 校验值

```
6105742b764ef76b6df547336e39eefb5194c2988ebd0d025a71957d9f457431  QuickCharge-v2.9.45-c.zip
```

```sh
sha256sum QuickCharge-v2.9.45-c.zip
```

这个包的构建是可复现的：同一份源码、同一套打包脚本，任何时候重打都得到相同的字节和相同的
sha256（`pack.sh` 里统一了 zip 内时间戳并去掉了高精度时间字段）。

## 源码

`bin/qcmon` 是编译产物，不是加密产物。源码 `src/qcmon.c`（2317 行纯 C99）、
构建脚本 `build-qcmon.sh`、49 项测试 `tests/c-test.sh` 都在仓库里，可自行核对与重建。

二进制里没有藏东西：

```sh
# 中文字符串要加 -e S，否则 strings 只认 ASCII，什么都看不到
strings -e S bin/qcmon | grep 伪装
# sysfs 路径、阈值也是明文
strings bin/qcmon | grep -E 'oplus_chg|thermal_zone' | head
# 反汇编是正常的 ARM 指令，不是压缩包也不是壳
aarch64-linux-gnu-objdump -d bin/qcmon | head -50
```

## 另一个版本

同版本另有 shell 版 `QuickCharge-v2.9.45.zip`（本次只改了一行文案）。
两版功能等价、可随时互换，详见 [QuickCharge](https://github.com/XiaohaoQvQ/QuickCharge) 的 README 与 Release。

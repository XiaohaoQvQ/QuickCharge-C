# Quick Charge v2.9.54-c（C 守护版）

> 对应包：`QuickCharge-v2.9.54-c.zip` · `versioncode 29541`

修两个日志缺陷，来自一份 26 分钟的真机日志。温控、充电策略没有改动，shell 版做了同样的修改。

## 一、未充电时快照刷屏

**现象**：灭屏待机 26 分钟写了 52 条快照，几乎全是这样：

```
22:04:15 vusb=0.00V imax=0.50A vbat=4.341V ibat=0mA 100% 33℃ ... Discharging
22:04:47 vusb=0.00V imax=0.50A vbat=4.366V ibat=0mA 100% 33℃ ... Discharging
```

**原因**：判重比整行字符串，`vbat` 抖几十 mV 就算"变化"，"未充电且不变不写"永远不成立。

**改法**：判重分两套。

| 状态 | 判据 |
|---|---|
| 充电中 | 整行比较 |
| 未充电 | 只比状态签名：状态、电量（5% 归并）、温度（2℃ 归并）、vusb、imax、type、cool、vooc、tech、屏幕 |

## 二、`ibat` 恒为 0mA

**现象**：快照里电池电流永远是 0，面板却显示 5-10W。

**原因**：快照读 `power_supply/battery/current_now`，灭屏后不刷新；面板走 `bcc_parms`。

**改法**：新增 `bcc_current()`，按 19 段逗号分隔解析 `bcc_parms` 第 9 段（mA），
再按优先级回退：

```
bcc_parms → battery/current_now → bms/current_now → battery/current_avg
```

这是 C 版**第一次**读 bcc——之前完全没有这段解析，双电芯机器上的电流只能靠
`current_now`，灭屏就是 0。

## 测试

`c-test` 67 通过，其中新增两条：

```
ok   current_now=-3mA 时用 bcc 的 -350mA
ok   灭屏 current_now=0 时用 bcc
```

其余套件：`chg-unit` 22、`cool-unit` 9、`zone-rank` 6、`mount-unit` 4、`sim` 621。

## 升级

直接覆盖安装，配置、日志、已调参数全部保留。

- 2.9.53-c → 2.9.54-c：`versioncode` 29531 → 29541

## 文件

| 项 | 值 |
|---|---|
| 大小 | 405,168 字节 |
| 文件数 | 20（含 `bin/qcmon`） |
| 最低要求 | Magisk / KernelSU / APatch 任一 |
| 架构 | arm64-v8a |

## 校验值

```
1aae740fcb30af6b06c0de59265a280a5c8cb6cb6dd6d1c4ffa286ad98dca2d6  QuickCharge-v2.9.54-c.zip
```

## 另一个版本

同版本另有 shell 版 `QuickCharge-v2.9.54.zip`，逻辑改动完全一致，详见
[QuickCharge](https://github.com/XiaohaoQvQ/QuickCharge)。

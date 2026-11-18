# Quick Charge v2.9.55-c（C 守护版）

> 对应包：`QuickCharge-v2.9.55-c.zip` · `versioncode 29551`

解决「打游戏只有 5~10W」。根因是降流解除被屏幕判定锁死：判成灭屏就直接
return，`cool_down` 停在驱动给的 7~13 没人清。shell 版做了同样的修改。

## 一、现场

2 小时 15 分的真机日志（PJD110，10:52 插电 5%，12:42 拔 95%）：

| 项 | 全程读数 |
|---|---|
| `scr` | **一直是 0** |
| `cool` | 5 / 7 / 8 / 10，一次 0 都没有 |
| `ibat` | −1 ~ −5mA |
| `vusb` / `imax` | 8.2~9.0V / 2.00A |

人在用手机，屏幕不可能是灭的——判定错了。`cool_guard` 开着，但
「降流解除：cool_down」一条日志都没有，说明写节点那一行从未执行到。

## 二、改法

**1. `cool_tick` 去掉屏幕依赖**

灭屏时驱动本来给 0，写 0 是空操作；只有亮屏被抬到 7~13 才需要干预。
温度保护保留：电池 ≥45℃ 写 2，≥48℃ 交还驱动。

**2. 亮度节点取最大值**

新增 `screen_scan()` 一次收集所有 `*/actual_brightness` 与 `*/brightness`，
`screen_sample()` 取其中的最大值，**全部为 0 才算灭屏**。原来的做法是
"取第一个存在的节点"，PJD110 的 `panel0-backlight/actual_brightness` 恒为 0，
排在最前就被选中，于是永远判灭屏。

**3. `next_tick` 充电即 3s**

原来是 `charging && g_screen_on` 才压到 2s，现在只要充电且 `cool_guard` 就
压到 3s，不再看屏幕。

**4. `chg_current` 的可信度**

`bcc_parms` 第 9 段小于 50mA 时视为噪声，继续比 `current_now` /
`bms/current_now` / `battery/current_avg`，取绝对值最大的。

**5. 启动记一条屏幕判定**

```
I [daemon] 屏幕 判定=灭 节点数=1
```

## 三、测试

`c-test` 67 通过，其中「灭屏 → 不干预」按新语义改为「灭屏也解除」。

其余套件（shell 侧）：`cool-unit` 12、`chg-unit` 23、`zone-rank` 6、
`mount-unit` 4、`sim` 621。

## 四、升级

直接覆盖安装，配置、日志、已调参数全部保留。

- 2.9.54-c → 2.9.55-c：`versioncode` 29541 → 29551

## 五、文件

| 项 | 值 |
|---|---|
| 大小 | 406,301 字节 |
| 文件数 | 20（含 `bin/qcmon`） |
| 最低要求 | Magisk / KernelSU / APatch 任一 |
| 架构 | arm64-v8a |

## 六、校验值

```
834a5728d03d26ee3af4efc1d495ec0fd1ba0cdb0b2dcb0ae597d941eefa3e7e  QuickCharge-v2.9.55-c.zip
```

## 七、另一个版本

同版本另有 shell 版 `QuickCharge-v2.9.55.zip`，逻辑改动完全一致，详见
[QuickCharge](https://github.com/XiaohaoQvQ/QuickCharge)。

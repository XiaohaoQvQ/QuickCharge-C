# Quick Charge v2.9.59-c（C 守护版）

> 对应包：`QuickCharge-v2.9.59-c.zip` · `versioncode 29592`

这一版没有功能改动，只修一份真机日志暴露出来的三处读数问题。
它们不影响充电本身，但会让日志和面板说假话。

## 一、`vbat=0.004V`：单位没归一

日志里电池电压整段是 `0.004`，电量却在 71% 正常爬升。

`power_supply/battery/voltage_now` 这个节点各机型不统一：多数报 µV
（`4219000` = 4.219V），也有报 mV 的（`4000` = 4.0V）。`chg_line()` 一律按 µV
交给 `fmt_uv()`，于是 4.0V 被算成 0.004V。

连带的后果比显示错更麻烦——判重签名里的功率是这么算的：

```c
pwk = ibm * vbat / 1000000000L;   /* 4400 * 4000 / 1e9 = 0 */
```

`vbat` 小 1000 倍，`pw=` 恒等于 `0W`，2.9.57 的「按 1W 分档判重」在这类机器上
直接失效，日志退回靠 `usb=8.55→8.96` 这种 0.01V 抖动触发，3 秒一条。

**改法**：读出来先归一。

```c
static long uv_norm(long v)
{
    if (v <= 0 || v >= 20000) return v;
    long t = v * 1000;
    if (t >= 100000 && t <= 30000000) return t;
    return v;
}
```

`vbat`、`vusb`、`imax` 一起走这个函数。阈值取 20000 是为了不误伤空载的
`30000`（0.03V），再用结果区间做二次校验。

## 二、日志时间比本地早 8 小时

同一份日志里两种时间：

```
11-19 09:52:27 | boot        ← 脚本，本地时间
11-19 01:53:18 qcmon 启动     ← qcmon，UTC
```

qcmon 是静态二进制，拿不到 Android 的时区库（设备上没有
`/usr/share/zoneinfo`），`localtime_r()` 直接落到 UTC。

**改法**：启动时问一次 `date +%z`，把 `+0800` 拼成 POSIX 的 `XXX-8`，
`setenv("TZ")` 之后 `tzset()`，不依赖 tzdata 文件。

## 三、屏幕判不出来

`屏幕 无亮度节点，按亮屏处理` —— `g_bl` 固定 `/sys/class/backlight`、
`g_fb` 固定 `/sys/graphics/fb0/blank`，这台两处都为空，屏幕状态于是恒为亮屏。

后果是「亮屏快充」永远停在「还没有基准值」，熄屏 30 秒延迟释放也不会触发。

**改法**：`backlight` 目录扫不到时再试几个常见落点 ——
`/sys/class/leds/lcd-backlight/actual_brightness`、
`/sys/class/leds/lcd-backlight/brightness`、`/sys/class/leds/lcd/brightness`、
`/sys/class/backlight/panel0-backlight/brightness`；`fb0/blank` 也补上
`/sys/class/graphics`、`/sys/devices/virtual/graphics` 两条路径。

## 四、测试

| 套件 | 结果 |
|---|---|
| `c-test` | 69 通过（新增「vbat 报 mV 时归一」） |
| `chg-unit` | 33 通过 |
| `cool-unit` | 16 通过 |
| `mount-unit` | 4 通过 |
| `zone-rank` | 6 通过 |
| `sim` | 621 通过 |

另外用临时根做过一次手工验证：`voltage_now=4000` 时快照输出 `vbat=4.000V`；
`/sys/class/backlight` 为空、`leds/lcd-backlight/brightness=512` 时 `scr=1`；
把 `date` 换成返回 `+0800` 的假脚本，守护日志时间跟着变成 UTC+8。

## 五、文件

| 项 | 值 |
|---|---|
| 最低要求 | Magisk / KernelSU / APatch 任一 |
| 架构 | arm64（含 C 守护） |

同版本 shell 版见 [QuickCharge](https://github.com/XiaohaoQvQ/QuickCharge)。

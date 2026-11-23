# Quick Charge v2.9.74-c（C 版）

> 对应包：`QuickCharge-v2.9.74-c.zip` · `versioncode 29630`

核心参数卡底部新增「极限充电」按钮，C 守护同步支持 `max_mode`。

## 一、按钮做什么

一次打开：温度墙 = 电池熔断阈值 − 1℃（默认 47℃）、限流提前量 0、档位 10、
快充阈值 85%、仅充电生效、停 horae、息屏不 +2℃。

开启前的 `wall` / `curr_edge` / `aggr` / `fast_cap` / `temp_mode` / `horae`
写入 `run/max.bak`，关闭时原样恢复。熔断阈值本身不改。

## 二、C 守护改动

- `conf_t` 增加 `max_mode`，启动时从 `state.conf` 解析。
- `curr_target`：`max_mode=1` 时墙取 `battery_limit - 1`，跳过息屏 +2 与 `lim-3` 钳制。
- `shell_target`：屏障同样取 `lim - 1`，伪装上限跟着抬高。
- 与 shell 版共用同一份 `lib/qc.sh` 与 `action.sh`，行为一致。

## 三、测试

| 套件 | 结果 |
|---|---|
| `c-test` | 86（新增 [6c]：42℃ 墙 → 47℃，关闭回 42℃）|
| `chg-unit` / `cool-unit` | 44 / 20 |
| `state-unit` | 12 |

## 四、边界

电量 80% 以上恒压、非原装头线只给 9V/1A，这两种情况按钮也拉不回来。

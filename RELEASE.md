# Quick Charge v3.1.4-c

**C 守护这一版补完了三个回流的写入 bug，并和 shell 版真正对齐。**

3.1.2 起 shell 包误带了 `bin/qcmon`，官方 shell 版实际跑的就是这份 C 守护——
所以下面这三个问题，3.1.2/3.1.3 的 shell 用户也一起中了。3.1.4 起 shell 包不带二进制，
但 C 包自己的 bug 必须单独修干净。

## P0：不再把 `usb/current_max` 当 FCC 目标

`scr_fcc_node` 的候选里一直留着 `usb/current_max`。那是 USB 端口/协议能力上限，
往里写会让充电器重新协商：实测 Find X8 60W→37W、一加 12 停 33W。3.0.4 在 shell 侧
就删了这个候选，C 版漏了整整十版。

现在与 shell 的 `sd_scr_is_input` 对齐：`/usb/`、`/charger/`、`/pc_port/`、
`/wireless/`、`input_current_limit` 全部排除，宁可判「本机不支持」。

## P0：息屏不再回写 FCC

旧注释写的是「FCC 路径不再放手」，等于复刻 2.9.73 的老 bug：驱动在息屏边界刚把
FCC 放开，模块又按历史峰值按回去，电池一热就满功率/10W 反复跳。

现在与 shell 一致：息屏只 `scr_learn()` 学基准，不写任何电池侧节点；内核参数
`ignore_screen_state` 这条在息屏时还原给驱动。同时 `scr_release` 补上「驱动已经放宽
就别拿旧值压回去」的判断。

## P1：`cool_down` 写入加电流前置与回滚

- 电流读不到 → 不写（过去 `chg_current()` 返回 0 被当成 0A，等于无条件写）
- `|Ibat| >= 2.5A` → 不写，不看档位
- 写 0 后记下原档位，下一轮复查：电流没上升就写回并冷却 5 分钟

## P1：参数与 shell 对齐

| 项 | 旧 | 新 |
|---|---|---|
| 默认 `shell_temp` | 25 | 30 |
| 温度范围 | 25-60 | 20-33 |
| 亮屏基线上限 | 8000 mA | 12000 mA |
| `cool_down` | 纯 2.5A 闸 | 同 shell（含回滚） |
| `qcmon -v` 版本串 | `3.0.12` | `3.1.4`（由 `module.prop` 注入） |

版本串这一条看着小，实际很影响排查：二进制自报 3.0.12，用户和我们都分不清
手上跑的是哪一版逻辑。

## 与 shell 版共用同一套脚本

`lib/qc.sh`、`customize.sh`、`action.sh`、`service.sh` 与 shell 版逐字节一致，
只有 `module.prop` 的 `channel=c` 和 `bin/qcmon` 不同。所以 3.1.4 里的这些也一并生效：

- `overrides on|off` 配置改写总开关
- 充电曲线列语义校验（首列不是 0.1℃ 就跳过）
- QEGA 取样点只在 `batt_spoof` 打开时替换
- `/proc/shell-temp` 关闭时写中性 30℃
- 迁移不再覆盖用户手改的伪装温度
- 每目录命中数日志 + 产物自检

## 测试

仿真 **604**、单元 **44 + 31 + 12 + 4**、c-test **84**，全过。
c-test 新增：只有 `usb/current_max` 时不接管、息屏 0 次写 FCC、`cool_down` 电流未知
不动、写 0 后电流没升写回。

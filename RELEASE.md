# Quick Charge v3.1.0-c

与 shell 版 3.1.0 同一份改动：**安全保护整套删除，只留充电速度**。C 守护与 shell
守护的逻辑保持一致，两版可以随时互换。

## 一、安全保护整套删除（破坏性）

熔断守卫、温度墙、温差档位 `aggr`、绝对电流墙 `curr_max`、电量阈值 `fast_cap`、
阶梯解锁的 45℃ 撤销、电池伪装的 46℃ 撤销——全部删除。

壳温现在是**恒定**上报：不看真实读数、不设温度墙、不设熔断线、不按电量退出。
`shell_temp` 默认 25℃，范围 20 到 33℃。

`src/qcmon.c` 里对应删掉 `sentinel_guard`、`guard_one`、`guard_cooled`、`is_guard_src`、
`limit_for_type`、`wall_of`、`curr_target`、`curr_keep`、`curr_is_ua`、`curr_read`、
`batt_capacity`、`batt_max`、`now_sec` 等 14 个函数和 8 个配置字段，2969 行降到 2659 行，
`gcc -Wall -Wextra` 零告警。

## 二、cool_tick 补上电流闸（P1）

C 版此前是裸写：只要 `cool_down` 非 0 就写 0。shell 侧 3.0.6 起就要求
「实测 `|ibat| < 2.5A` 才写」，因为写入本身会让 OPlus 重选档——实测 4.3A 时写一下
就从 32W 掉到 6W。现在两边对齐，`chg_current()` 同时补了单位归一
（`0 < |v| < 20000` 判为 mA），报 mA 的机型 `3000` 不会被当成 3mA 而误判「已降流」。

## 三、顺带修的几处

- `scr_apply()` 修掉 `raw` 变量遮蔽，并复用为判断单位已读过的节点原文，不再重复 `read_line`。
- `screen_sample()` 内层循环变量改名，不再遮蔽外层 `i`。
- `shell_target()` 改成恒定值（20 到 33℃ 夹取），真实温度只写遥测。
- 默认 `shell_temp` 与 shell 版一致，都是 25℃。

## 版本号

`3.1.0-c` / `versionCode 301000`，shell 版 `3.1.0` / `versionCode 301001`。

## 验证

- `tests/c-test.sh` 81 项全过。新增：高温继续伪装（电池 50℃ / 板 90℃ 都不撤销）、
  降流解除的电流闸四条（4.3A 不动、报 mA 的 3A 不动、2499mA 写 0、边界 2.5A 含在闸内）。
- 单元测试（chg / cool / state / mount）共 87 项全过。
- 二进制 `bin/qcmon` 为 arm64 静态链接，`strings` 已确认不含 `sentinel_limit`、
  `battery_limit`、`curr_max`、`cool_guard`、`wall` 等已删配置项。

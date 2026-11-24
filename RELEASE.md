# Quick Charge v3.0.6-c

与 shell 版 3.0.6 同步。用户反馈「只剩十几 W、越来越低」，而日志显示这台机
`charge-nodes=1`、`fcc 节点=none`——3.0.5 的 FCC 修复对它根本不生效，模块没有写
任何电流节点。真正掉档的是**写 `cool_down` 这个动作**。

## 证据（PKB110 / MT6991 / 3.0.5-c）

```
19:34:35  vusb=8.74V imax=2.00A ibat=-4277mA pb=18.1W
19:34:57  降流解除：cool_down 2 → 0（电池 38℃ 电量 60%）
19:35:00  vusb=8.63V ibat=-2337mA pb=9.7W     ← 三秒后掉档
19:36:53  vusb=8.73V ibat=-136mA  pb=0.6W
```

`pb` 是净入电池功率（亮屏耗电会吃掉），`8.7V/2A` 是 OPPO 亮屏 VOOC 的常规输入档。

## 根因

`cool_down` 写入会让 OPlus 重新选充电档，不管写 0 还是 1：

- PKB110：`ibat` 已 4.3A，仍被写 `2 → 0`，三秒后 18W 掉到 9.7W；
- PKX110：写 `0 → 1` 后 `charge_technology` 7→14、3.6A→1.0A、32W→9W。

旧逻辑只看"当前值 ≠ 目标值"，不看此刻电流是否正常，于是好机器也被写掉一档。

## 修复

- `cool_tick` 写 `cool_down=0` 前先读 `chg_current()`：**≥2.5A 视为未降流，直接返回**；
- 写 0 的日志带上电流（`电流 %ldmA`）；
- `chg_input()` 补 `charger/current_now`、`battery/input_current_now`，并兜底读
  `oplus_chg/charger/current_now`、`oplus_chg/current_now`——欧加把输入电流挂在这里，
  所以以前 `iusb`/`pin` 一直是 `?`；
- `bin/qcmon` 重编为 3.0.6。

## 版本号

`3.0.6-c` / `versionCode 300012`。`updateJson` 仍指向 C 仓。

## 验证

C c-test 94 项全过；新增「电流 4A 已正常：不碰 cool_down」「电流 1A 被降流：写 0 解除」。

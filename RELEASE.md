# Quick Charge v3.1.8-c

C 守护这版有**实质改动**：`_pa` 温区分类对齐 shell 版。其余与 shell 版同步（删死代码、改文案）。

## 一、`wifi_pa` 不再被 `noncore` 伪装（唯一行为改动）

shell 的 `sd_zone_class` 有 `*_pa` 规则判 `board`，C 的 `cls_of` 的 board 分支只认精确的 `pa`，于是 `wifi_pa` 落到 `C_OTHER`。而 `in_scope(SC_NONCORE)` 只排除 core/shell/battery/board：

```c
case SC_NONCORE:
    if (c == C_CORE || c == C_SHELL) return 0;
    if (c == C_BATTERY || c == C_BOARD) return 0;
    return 1;                  /* C_OTHER 漏进来 */
}
```

结果：**开 `noncore_temp` 时 C 版会把 WiFi PA 温度一起钉死，shell 版不会**。射频高负载时系统会误判散热余量——不掉功率，但可能加剧发热。

修法是 3 行，放在 core 判断**之后**（`cpu_pa` 先被 core 命中，与 shell 的匹配顺序一致）：

```c
size_t n = strlen(t);
if (n >= 3 && !strcmp(t + n - 3, "_pa")) return C_BOARD;
```

`c-test` 新增 `[17b]` 六条：分类走 `zones`（`wifi_pa → board`、`cpu_pa → core`），行为走完整 daemon（`wifi_pa` 保持 46℃、`pa` 保持 39℃、`bms` 仍是 other 被伪装、`cpu_pa` 保持 50℃）。

反向验证：去掉这 3 行重编，`wifi_pa 分类错：cls=other` 与 `不被 noncore 伪装（期望 46000，实际 30000）` 两条立刻红。

## 二、删掉熔断读数链（-86 行）

`sd_batt_best_v` → `sd_batt_guard_v` → `sd_guard_read_v` → `sd_sentinel_best_v`。熔断在 3.1.0 就删了，这四个函数生产零调用，只剩 `sd_batt_guard_v` 里一段「为什么熔断要取高值」的注释在骗人。

C 侧本身没有这条链（读 `power_supply` 走 `batt_real_v`），删的是同步过来的 `lib/qc.sh`。

## 三、文案

- `unlock` 不再承诺「电池 ≥45℃ 自动撤销」—— 代码里没有这个逻辑，`SD_UNLOCK_HARD` 零引用。4 处统一改成「不按温度撤销，要还原得手动关」。
- 安装提示不再建议「觉得慢就调回 25℃」（25℃ 正是低温限流区间），改成 `base 32` 并写明代价。
- `TECH.md` 的「极限功率」整节前提已不存在（熔断、46℃ 硬上限、120 秒冷却、拒绝开启，四条全无），标题与两段重写；亮屏快充条件表里「熔断未触发 / ≤46℃ / ≥47℃ 不接管」三行也早过时，3.1.3 起不看电池温度。
- `action.sh` 帮助与菜单里两处「仅充电 25℃」其实是 30℃，改正。

## 测试

| 项 | 结果 |
|---|---|
| 语法 | 7/7 |
| 单元 | 92（state 12 / cool 28 / mount 4 / chg 44 / zone-rank 4） |
| 仿真 | 613 |
| c-test（x86 现编） | 93 |
| c-test（官方 arm64 bin，qemu） | 93 |

`bin/qcmon` 重编，自报 `3.1.8`，md5 `b76d3c62`。用例数从 87 变 93 是新增 6 条 `_pa` 断言；单测 97 → 92 是删掉守死功能的 5 条。

## 装机

覆盖安装，重启即可。与 shell 版随时互换，`state.conf` 是同一份。

**这版唯一的用户可见变化**：开了 `noncore_temp` 的人，PA 温度不再被伪装。没开这个开关的，行为不变。

- 完整包：`QuickCharge-v3.1.8-c.zip`
- 更新清单：`update.json`

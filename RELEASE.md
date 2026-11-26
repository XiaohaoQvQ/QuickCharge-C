# Quick Charge v3.1.5-c

## 3.1.4-c 是空发布，这版才是真的

上一版 `src/qcmon.c`、`bin/qcmon`、`tests/c-test.sh` 三项与 3.1.3-c 逐字节一致：
脚本层同步了，守护层的四个修复一条都没编进二进制，版本号却照升。
装 3.1.4-c 的人拿到的是 3.1.3-c 的充电行为。

这版三件一起同步重编，二进制特征串逐项核对：

| 特征串 | 3.1.4-c | 3.1.5-c |
|---|---|---|
| `/pc_port/`（输入侧节点排除） | 0 | 1 |
| `/wireless/` | 0 | 1 |
| `降流解除反噬`（cool_down 回滚） | 0 | 1 |
| `息屏保持（回写 FCC`（旧息屏分支） | 1 | 0 |

`qcmon -v` 报 `3.1.5`，`c-test` 从 81 项扩到 84 项。

## C 守护这版真正生效的修复

**息屏不再回写 FCC。** 旧代码在息屏时仍然把亮屏学到的基准写回 `constant_charge_current_max`，
驱动刚在息屏边界把电流放开，模块又踩一脚刹车 —— 实测 60W 掉到 30W。现在息屏只学基准、不写节点。

**不再把 `usb/current_max` 当 FCC 目标。** 那是 PD 协商出来的充电器能力值（常见 2A），
写它等于把目标压到 2A。候选表删掉这一项，并新增 `scr_is_input` 排除
`/usb/`、`/charger/`、`/pc_port/`、`/wireless/`、`input_current_limit`。

**`cool_down` 不再裸写。** 电流读不到不动；`|Ibat| ≥ 2.5A` 不动；写了 0 之后下一轮复查，
电流没升就写回原档位并冷却 5 分钟 —— 写这个节点本身会让 OPlus 重选档。

**`scr_release` 不再拿旧值压回去。** 驱动已经放宽（息屏边界/温控回收）时，
按存档还原会把电流又拉回低档。

**参数与 shell 对齐。** 默认 30℃、范围 20-33、亮屏基线 12000mA；
版本串由 `module.prop` 注入，不再自报 `3.0.12`。

## 脚本层同步

`lib/qc.sh`、`customize.sh`、`action.sh`、`service.sh`、`post-fs-data.sh` 与 shell 版同源，
因此 shell 版这几项修复也一并生效：

- 伪装温度三处兜底 `25` → `30`；
- `sd_state_get` 补 `shell_temp_manual`、`overrides_off` —— 3.1.4 里这两个键读不出来，
  手改标记与 `overrides` 开关实际无效；
- 关闭伪装时不再改写 `/proc/shell-temp`（原来写死 30℃，节点永久停在假值）；
- 迁移提示改按 `defrev` 判断，3.1.3 被无条件改成 30 的用户终于能看到"可试回 25"；
- 曲线改写放宽：允许从 0℃ 起、允许降序，仍拒绝 ℃ 整数与电压首列；
- `overrides off` 时 `post-fs-data` 整段跳过；
- 包内不再带 `docs/`（74 KB）。
- `/my_stock/etc`、`/my_product/etc` 纳入配置扫描（ColorOS 14+ 的温控表在这两个分区）。

## 测试

仿真 **613**、单元 **44 + 31 + 12 + 4**、c-test **84**，全过。

这次 `c-test` 在 C 仓自己的源码上跑，不是拿主仓源码代跑。

## 装机

覆盖刷入，参数与日志保留。C 包 `channel=c`，会走 `bin/qcmon`。

- 完整包：`QuickCharge-v3.1.5-c.zip`（406 KB）
- 更新清单：`update.json`

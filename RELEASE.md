# Quick Charge v2.9.66-c（C 版）

> 对应包：`QuickCharge-v2.9.66-c.zip` · `versioncode 29614`

与 shell 版同步：「真实」格下只写温度点；协议判定改成 `tech` 优先。

## 一、真实读数来源

`守护读数 · batt` / `现场直读 · batt` → 直接显示 `batt`。

## 二、协议判定顺序

欧加私有协议在线时 `usb/type` 也可能写 `USB_PD`，旧逻辑先看类型串，直接判 PD 并返回，
`tech=8` 永远轮不到。新顺序：

1. `charge_technology` 1~13 → 私有
2. 类型串 VOOC / DART / Flash → 私有（压过 `tech=14`）
3. `voocchg_ing=1` → 私有
4. 类型串 PPS / UFCS → 公有
5. `tech≥14` → 公有
6. 类型串 PD → PD
7. 协商 ≥8V 且 ≥5A → 私有
8. 其余 → `unknown`

C 端 `chg_proto()` 增加 `rtype` 参数，`chg_line` 与 env 头都读 `usb/real_type`，
日志行加 `rtype=`。

## 三、验证

新增 `type=USB_PD + tech=8 → vooc`、`type=USB_PD 无 tech → pd` 等用例；
`c-test` 83、`chg-unit` 53、`cool` 16、`mount` 4、`zone` 6、`sim` 621 全通过。

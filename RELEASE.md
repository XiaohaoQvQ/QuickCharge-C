# Quick Charge v3.0.2-c

C 守护与 shell 版同步的 3.0.2 正式版。守护逻辑与 3.0.1-c 完全一致，本版只接入在线升级。

## 在线升级

`module.prop` 加了 `updateJson`，指向本仓根目录的 `update.json`：

```properties
updateJson=https://cdn.jsdelivr.net/gh/XiaohaoQvQ/QuickCharge-C@main/update.json
```

```json
{
  "version": "3.0.2-c",
  "versionCode": 300004,
  "zipUrl": "https://github.com/XiaohaoQvQ/QuickCharge-C/releases/download/3.0.2-c/QuickCharge-v3.0.2-c.zip",
  "changelog": "https://cdn.jsdelivr.net/gh/XiaohaoQvQ/QuickCharge-C@main/RELEASE.md"
}
```

管理器拉这个 JSON，`versionCode` 大于本地 `versioncode` 就提示更新，点一下自动下载刷入，
流程与手动刷包一致。

## C 版的 updateJson 必须指 C 仓

两仓模块 ID 同为 `quickcharge`，互为覆盖关系。如果 C 版的 `updateJson` 指到 shell 仓，
用户点更新会装进 shell 包，`bin/qcmon` 消失、守护退回 shell 实现，看起来像"更新后变卡"。
这次两仓各配一份，互不指向。

## 这一版仍需手动刷一次

3.0.1-c 及更早的 `module.prop` 没有 `updateJson`，管理器不会去查远端，收不到提示。
手动刷入 3.0.2-c 之后通道才打通。

## 版本号

`3.0.2-c` / `versioncode 300004`。C 用偶数通道，与 shell 版 `300005` 互不覆盖。

## 验证

c-test 90 项、`chg-unit` 44、`cool-unit` 20、`state-unit` 12 全过；`bin/qcmon` 重编为
arm64 静态版，版本串 `3.0.2`。`update.json` 的 `versionCode` 与 `module.prop` 核对一致。

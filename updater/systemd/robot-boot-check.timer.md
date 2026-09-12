# robot-boot-check.timer 文件解析

## 文件位置

`d:\microduck\updater\systemd\robot-boot-check.timer`

## 设计决策

用**截止时间**而非每个 daemon 的 `OnFailure=`。所有 daemon unit 用 `Restart=always`（各自 unit 文件论证），且未覆盖 systemd 默认启动限制——所以 btd 每 5 秒重启会无限 crash 循环而永不进入 `failed`。让 `OnFailure=` 触发需重调这些 unit 使 daemon 放弃，这会为救援情况降级正常情况。

截止时间方式无需改 unit，捕获永远重启者，问的是对的问题——"这个发布起来了吗？"而非"systemd 放弃了什么吗？"。

## [Timer]

| 字段 | 值 | 说明 |
|---|---|---|
| `OnBootSec` | 180 | 180 秒：板子比看起来慢（hci0 ~73s 才出现），daemon 等硬件不退出，慢启动不是失败 |
| `AccuracySec` | 10s | 宽松定时器不唤醒 |

## 关键摘要

robot-boot-check.timer 在启动 180s 后触发启动检查：用截止时间而非 OnFailure（因 daemon 用 Restart=always 永不进入 failed）；180s 给慢硬件启动留余量；宽松 AccuracySec。

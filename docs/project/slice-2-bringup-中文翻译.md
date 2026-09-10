# 硬件上的 Slice 2 启动

在真实机器人上运行 slice 2 的笔记。以下所有内容都是在 Radxa Zero 3W 上观察到的，而非推断。

## 状态

Slice 2 已合并，板上发现的唯一问题——**`ort` 崩溃而非返回错误，这杀死了控制线程并使 `robot.health` 归咎于错误的东西**——已修复：`duck-control::policy::catching_ort_panics` 将其转换为 `PolicyError`，因此它走现有的"保持姿势并报告原因"路径。

**两者现在都在硬件上关闭。** 修复通过 `ORT_DYLIB_PATH` 将 `robotd` 指向 1.20.1 运行时在板上进行了验证：循环继续滴答，健康状态指出了版本不匹配，而不是一个死掉的控制线程报告"尚未完成一个周期"。然后 slice 2 在滴答中运行推理，`0.2.0`——第一个包含其中任何内容的发布——从稳定通道安装。

没有任何东西再链接到这个文件。它被保留为板上实际说了什么的记录；可复用的一半是下面的配方，而有人再次需要的命令应该最终出现在 [`cheatsheet.md`](../robot/cheatsheet.md) 中。

## 已经能工作的，已在板上验证

在有线机器人上运行 `0.1.4`（slice 1）：

- 15 个舵机和 `imu_to_dxl` 板在 `/dev/ttyS2` 上应答
- 控制循环在 **50.0 Hz**，15022 个滴答中 `missed=3`（0.02%）
- `robotctl health` → `healthy`
- 更新路径端到端演练：安装、健康门、提交和自动回退

因此总线、舵机、IMU、速率和更新器**不是**嫌疑人。现在任何失败的都是 slice 2 的代码或板上的 ONNX Runtime。

## 失败

`sudo robotctl update apply daemon --ref slice-2-walk-stand` 回退了：

```
  HealthGate
  RollingBack
{
  "attempted": "0.1.4-dev.58.6781f98",
  "outcome": "rolled_back",
  "reason": "health check failed: not healthy within 30s:
             control loop has not completed a cycle yet",
  "reverted_to": "0.1.4"
}
```

日志给出了真正的原因：

```
thread 'control' panicked at ort-2.0.0-rc.11/src/lib.rs:191:41:
Failed to load ONNX Runtime dylib: Error { code: GenericFailure, msg:
  "ort 2.0.0-rc.11 is not compatible with the ONNX Runtime binary found at
   `libonnxruntime.so`; expected version >= '1.23.x', but got '1.20.1'" }
```

### 为什么健康原因没用

`RobotState::health` 在 `ticks == 0 && startup_bus_failures == 0` 时报告 `control loop has not completed a cycle yet`。一个崩溃的控制*线程*不会杀死进程，因此 `robotd` 保持运行，继续服务其 socket，并以恰好那个——那个不命名任何原因的消息——回答。把原因字符串读为"循环从未启动且从未记录原因"，而不是"仍在启动"。

## 两个原因。一个已修复。

**1. 板上有一个不兼容的运行时——在 #17 中修复（已合并）。** `setup-board.sh` 固定了 ONNX Runtime 1.20.1；`ort 2.0.0-rc.11` 需要 >= 1.23。检查现在是版本感知的，因此重新运行脚本会替换错误版本而不是报告"已存在"。

下限和目标存在于根 `Cargo.toml` 的 `[workspace.metadata.onnxruntime]` 中，#18 从它们生成发布的 `hooks/preinstall`，因此低于下限的板被治愈——或者更新在*交换之前*中止——而不是安装然后崩溃。

**2. `ort` 崩溃而非报错——已修复。**

`ensure_runtime()` 在让 `ort` 接触它之前用 `libloading` 探测 dylib。它的文档注释曾经声称：

> 一个成功的探测意味着它的加载也会成功，崩溃不可能触发

**板上证伪了这一点。** 探测只证明库*能加载*；1.20.1 加载得很好。`ort` 自己的兼容性检查然后拒绝*版本*并在 `setup_api` 内部崩溃，这是 `ensure_runtime` 看不到的。因此守卫关闭了"缺失"情况而不是"错误版本"情况——而且按构造它也不可能关闭每一个未来的 `ort` 崩溃。

修复不试图这样做。`catching_ort_panics` 包裹 `Policy::load` 内部的 `ort` 调用，并将任何崩溃转换为 `PolicyError::RuntimePanic`，携带崩溃消息——上面的版本号就是整个诊断，因此丢失它们会留下一个没人能据此行动的健康原因。这把崩溃放在 `robotd` 已经为无法加载的策略准备的路径上：记录 `policy unavailable; holding the pose`，存储原因，将 `controller` 留为 `None`，**保持速率滴答**，并报告 `policy unavailable: <reason>`，以便更新器以一个陈述的原因回退发布。

关于它需要知道的两件事：

- 捕获只包裹 `ort` 工作，而不是所有 `load`，因此我们自己的真正 bug 不会被重新标记为"policy unavailable"。需要 `AssertUnwindSafe` 因为 `Session` 不是 `UnwindSafe`；会话在成功时被移入 `Policy`，在失败时被丢弃，因此我们的任何东西都不会在捕获后被观察。
- **`panic = "abort"` 会破坏它。** 根 `Cargo.toml` 中今天没有 `[profile.release]`；添加一个会静默地恢复死掉的控制线程。

### 测试覆盖了什么，以及没覆盖什么

离线覆盖，在 `duck-control` 和 `robotd` 中：`ort` 路径上的崩溃变成错误并保留其消息，成功不受影响地通过，不可打印的 payload 仍然产生一个原因，以及——通过 `an_unloadable_policy_holds_the_pose_and_reports_why`——一个无法加载的策略让循环继续滴答，底层原因在健康字符串中。

离线未覆盖：一个*真实的* `ort` 崩溃穿过控制循环。复现它需要一个错误版本的运行时，这是一块板，而在 `Policy::load` 内部伪造一个意味着在 `duck-control` 中交付一个故障注入旋钮来测试三行。下面的板上检查是关闭它的东西。

## 在板上验证

板需要一次开发密钥，否则 `--ref` 被拒绝。`install.sh` 做两半——安装密钥并翻转 `allow_dev_keys`——给定公钥的路径：

```bash
sudo DUCK_TOKEN="$DUCK_TOKEN" DUCK_DEV_KEY=/tmp/team.dev.pub sh /tmp/install.sh
```

`team.dev.pub` 提交在 `deploy/dev-key/`，在 `trusted_keys/` 之外，因此默认没有东西安装它。手动等效在 [`../deploy/README.md`](../../deploy/README.md) 中。

然后：

```bash
sudo robotctl update apply daemon --ref slice-2-walk-stand
```

**在信任 `setup-board.sh` 之前重新获取它。** `/usr/local/sbin/robot-setup-board` 是上次运行时复制的快照；它从不自我刷新，因此它可以静默地运行 #17 之前的逻辑并为不兼容的运行时报告"已存在"。

成功看起来像状态块中的 `ONNX Runtime  1.28.0`，更新提交而不是回退，以及：

```bash
journalctl -u robotd -b --no-pager | grep -E 'policy|control loop'
```

显示 `policy loaded` 后跟 `control loop running driving=true`。然后重新测量速率——slice 2 在同一个滴答中添加了推理，上面的 slice 1 基线（50.0 Hz，`missed=3`）是要比较的对象。`missed` 的大幅跳跃是推理成本，而不是抖动。

## 约定

- **在 `/tmp` 下的全新克隆中开分支**，永远不要在工作检出中。共享克隆中陈旧的工作树是 #13 静默回退 #12 的原因——`git checkout -b` 把未提交的更改带进新分支，`git add -A` 把它们作为删除提交。
- 提交尾部是 `Assisted-by: Claude:claude-opus-5`。永远不要用 `Co-Authored-By`。
- 范围测试运行：`cargo test -p <crate>`，一次。把 `--workspace` 留给 PR 前检查。
- 在做架构决定之前询问。
- 修复发布路径 bug 并切一个发布；不要交出本地变通方案。

## 故意未做

- MuJoCo 后端，以及剩余的六个技能。
- 每关节限制。`duck-control/src/safety.rs` 钳制到执行器行程（±π），而不是每关节范围；那需要 alpha MJCF 被 vendored。
- 来自 `microduck_brain` 的黄金观察向量，用于将 61-D 编码与原型钉住。布局测试覆盖形状，不覆盖与原始的一致性。
- `hooks/postinstall`——#18 只交付 `preinstall`。
#（注：内容由AI生成）

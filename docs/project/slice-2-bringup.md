# 切片 2 在硬件上的启动

在真实机器人上运行切片 2 的笔记。以下所有内容都在 Radxa Zero 3W 上观察到，而非推断。

## 状态

切片 2 已合并，板子发现的唯一一个问题 —— **`ort` 恐慌而非返回错误，这杀死了控制线程并让 `robot.health` 责怪了错误的东西** —— 已修复：`duck-control::policy::catching_ort_panics` 把它变成一个 `PolicyError`，因此它走上现有的"保持姿态并报告原因"路径。

**两者现在都已在硬件上关闭。** 该修复在板子上通过把 `robotd` 指向一个经 `ORT_DYLIB_PATH` 的 1.20.1 运行时来演练：循环保持滴答，健康命名了版本不匹配，而不是一个死掉的控制线程报告"尚未完成一个周期"。切片 2 随后在滴答中加入推理运行，`0.2.0` —— 包含其中任何内容的第一个版本 —— 从稳定通道安装。

没有任何东西再链接到本文件。它被保留作为板子实际说了什么的记录；可复用的一半是下面的配方，而有人再次需要的命令应该去 [`cheatsheet.md`](../robot/cheatsheet.md)。

## 已经工作的，在板子上验证

在一个有线机器人上运行 `0.1.4`（切片 1）：

- 15 个舵机与 `imu_to_dxl` 板在 `/dev/ttyS2` 上应答
- 控制循环在 **50.0 Hz**，15022 个滴答中 `missed=3`（0.02%）
- `robotctl health` → `healthy`
- 更新路径端到端演练：安装、健康门、提交与自动回滚

因此总线、舵机、IMU、速率与更新器**不是**嫌疑。任何现在失败的东西都是切片 2 的代码或板子的 ONNX Runtime。

## 失败

`sudo robotctl update apply daemon --ref slice-2-walk-stand` 回滚：

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

### 为什么健康原因无用

`RobotState::health` 在 `ticks == 0 && startup_bus_failures == 0` 时报告 `control loop has not completed a cycle yet`。一个恐慌的控制*线程*不会杀死进程，因此 `robotd` 保持运行，继续服务其 socket，并精确地用那个 —— 那个不命名任何原因的消息 —— 应答。把原因字符串读为"循环从未启动且从未记录原因"，而非"仍在启动"。

## 两个原因。一个已修复。

**1. 板子有一个不兼容的运行时 —— 已在 #17 修复（已合并）。** `setup-board.sh` 固定了 ONNX Runtime 1.20.1；`ort 2.0.0-rc.11` 需要 >= 1.23。检查现在是版本感知的，因此重新运行脚本会替换错误版本而非报告"已存在"。

下限与目标位于根 `Cargo.toml` 的 `[workspace.metadata.onnxruntime]` 中，#18 从它们生成版本的 `hooks/preinstall`，以便一个低于下限的板子被修复 —— 或更新在*交换前*中止 —— 而非安装然后恐慌。

**2. `ort` 恐慌而非报错 —— 已修复。**

`ensure_runtime()` 在让 `ort` 触碰动态库之前用 `libloading` 探测它。它的文档注释曾经声称：

> 一个成功的探测意味着它的加载也会成功，恐慌不会触发

**板子证伪了这一点。** 探测只证明库*能加载*；1.20.1 加载正常。然后 `ort` 自己的兼容性检查拒绝*版本*并在 `setup_api` 内部恐慌，`ensure_runtime` 看不见。因此该守卫关闭了"缺失"情况而非"错误版本"情况 —— 且按构造它也无法关闭每一个未来的 `ort` 恐慌。

修复不试图这么做。`catching_ort_panics` 把 `ort` 调用包装在 `Policy::load` 内部，并把任何恐慌转换为 `PolicyError::RuntimePanic`，携带恐慌消息 —— 上面的版本号就是全部诊断，因此丢失它们会留下一个无人能据此行动的健康原因。这把一个恐慌放在 `robotd` 已经为一个无法加载的策略准备的路径上：记录 `policy unavailable; holding the pose`，存储原因，把 `controller` 留为 `None`，**保持以速率滴答**，并报告 `policy unavailable: <reason>`，以便更新器以一个陈述的原因回滚版本。

关于它有两件事要知道：

- 捕获只包装 `ort` 工作，而非全部 `load`，因此我们的一个真正 bug 不会被重新标记为"policy unavailable"。需要 `AssertUnwindSafe` 因为 `Session` 不是 `UnwindSafe`；会话在成功时移入 `Policy`，失败时丢弃，因此捕获后不会观察到我们的任何东西。
- **`panic = "abort"` 会击败它。** 今天根 `Cargo.toml` 中没有 `[profile.release]`；添加一个会静默地恢复死掉的控制线程。

### 测试覆盖什么，以及不覆盖什么

离线覆盖，在 `duck-control` 与 `robotd` 中：`ort` 路径上的恐慌变成一个错误并保留其消息，成功原样通过，一个不可打印的载荷仍产生一个原因，以及 —— 通过 `an_unloadable_policy_holds_the_pose_and_reports_why` —— 一个无法加载的策略让循环保持滴答，把根本原因放在健康字符串中。

离线未覆盖：一个*真实的* `ort` 恐慌穿过控制循环。复现它需要一个错误版本的运行时，那是一块板子，而在 `Policy::load` 内部伪造一个意味着在 `duck-control` 中交付一个故障注入旋钮来测试三行。下面的板子检查才是关闭它的东西。

## 在板子上验证

板子需要一次 dev 密钥，否则 `--ref` 被拒绝。`install.sh` 做两半 —— 安装密钥并翻转 `allow_dev_keys` —— 给定公钥半的路径：

```bash
sudo DUCK_TOKEN="$DUCK_TOKEN" DUCK_DEV_KEY=/tmp/team.dev.pub sh /tmp/install.sh
```

`team.dev.pub` 提交在 `deploy/dev-key/`，在 `trusted_keys/` 之外，因此默认没有东西安装它。手动等价物在 [`../deploy/README.md`](../../deploy/README.md)。

然后：

```bash
sudo robotctl update apply daemon --ref slice-2-walk-stand
```

**在信任 `setup-board.sh` 之前重新获取它。** `/usr/local/sbin/robot-setup-board` 是上次运行时复制的快照；它从不刷新自己，因此它可以静默地运行 #17 之前的逻辑并对一个不兼容的运行时报告"已存在"。

成功看起来像状态块中的 `ONNX Runtime  1.28.0`，更新提交而非回滚，以及：

```bash
journalctl -u robotd -b --no-pager | grep -E 'policy|control loop'
```

显示 `policy loaded` 后跟 `control loop running driving=true`。然后重新测量速率 —— 切片 2 把推理加到同一个滴答上，而上面的切片 1 基线（50.0 Hz，`missed=3`）是要比较的对象。`missed` 的大幅跳升是推理成本，而非抖动。

## 约定

- **在 `/tmp` 下的一个全新克隆中分支**，绝不在工作检出中。共享克隆中一个陈旧的工作树正是 #13 静默回滚 #12 的方式 —— `git checkout -b` 把未提交的更改带进一个新分支，`git add -A` 把它们作为删除提交。
- 提交尾注是 `Assisted-by: Claude:claude-opus-5`。绝不要 `Co-Authored-By`。
- 限定测试运行范围：`cargo test -p <crate>`，一次。把 `--workspace` 留到 PR 前检查。
- 做架构决策前先问。
- 修复发布路径 bug 并切一个版本；不要交出一个本地变通方案。

## 故意未做

- MuJoCo 后端，以及剩余六个技能。
- 每关节限制。`duck-control/src/safety.rs` 钳制到执行器行程（±π），而非每关节范围；那需要 vendor 的 alpha MJCF。
- 来自 `microduck_brain` 的黄金观测向量，以对照原型固定 61-D 编码。布局测试覆盖形状，不覆盖与原始的一致性。
- `hooks/postinstall` —— #18 只交付 `preinstall`。

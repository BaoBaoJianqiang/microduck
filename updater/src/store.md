# store.rs 文件解析

## 文件位置

`d:\microduck\updater\src\store.rs`

## 定位

磁盘发布存储：版本化目录 + 消费者读取的 symlink。原子性来自单次 `rename(2)` 覆盖 symlink，绝不会有部分写入的发布处于活状态。

```
/opt/robot/daemon/
├── releases/1.4.1/   ← previous，保留用于回滚
├── releases/1.4.2/   ← new
└── current → releases/1.4.2
```

**此处不得触碰机器人特定状态**——校准、学习状态、用户配置在 `install_dir` 之外，使交换/回滚不销毁它们。

## 关键路径常量

- `STAGING_PREFIX = ".staging-"` — 不可能解析为 semver，防 list 把 staging 当真发布
- `RELEASES_DIR = "releases"`
- `CURRENT_LINK = "current"` — 消费者 symlink
- `GOLDEN_LINK = "golden"` — 已知良好发布的 symlink，供 `scripts/robot-rescue` 用单个 readlink 读取（无需解析 updater.toml）

## `Store` 方法

- `current()` — current 指向的版本，`Ok(None)` 表示无链接或悬空（可恢复）
- `golden()` — golden 链接
- `list()` — 已安装版本，**semver 排序**新→旧，忽略 staging
- `swap_to(version)` — 原子指向 current
- `mark_golden(version)` — 写 golden 链接（每次启动幂等重写）
- `prune(keep_previous, golden)` — 删除旧版本，保留 active + keep_previous 个 + golden（golden 不计入数量）
- `clean_staging()` — 清理中断运行的 staging 残留
- `available_space()` — fs4 取可用空间

### `link_to(name, version)` 原子性

1. 校验 release 目录存在
2. 在旁边创建 `.tmp` symlink（相对目标）
3. `rename(tmp, link)` — 同目录原子，读者只见旧或新
4. `fsync_parent(link)` — **持久性**，非仅原子性：boot counter 已在交换前 armed，若链接落盘而 pending 记录没落盘，断电会留坏发布活且无 trial 回滚

## 测试

- `swap_is_atomic_and_repeatable` — 升级与回滚路径
- `golden_is_published_independently_of_current` — 两链接互不干扰
- `marking_golden_refuses_a_release_that_is_not_installed` — 防悬空 golden
- `list_orders_by_semver_not_string` — 1.10.0 在 1.9.0 之上
- `prune_never_removes_golden` — golden 豁免于计数
- `clean_staging_removes_only_staging_dirs`

## 关键摘要

store.rs 管理版本化发布目录与原子 symlink 交换：`current` 用于活版本、`golden` 供救援脚本无需解析配置即可读取；swap 通过"建临时链接+rename+fsync 父目录"保证原子且持久；list 按 semver 排序；prune 永不删 golden；staging 用不可能的 semver 前缀防误判。

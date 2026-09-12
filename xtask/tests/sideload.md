# tests/sideload.rs 文件解析（xtask）

## 文件位置

`d:\microduck\xtask\tests\sideload.rs`

## 定位

检查 sideload 目录与读取它的 unit 之间的一致性——`scripts/dev-push.sh` 把 release 拷到板上，`updaterd` 安装它。两者靠约定就路径达成一致。

## 核心问题：`PrivateTmp=yes`

`updaterd.service` 设 `PrivateTmp=yes`，使该 unit 有自己的 `/tmp` **和** `/var/tmp`。因此这两个目录下的目录不是 daemon 看到的那个——push 成功、文件明明在，apply 却报 "no manifest for version ... in /var/tmp/duck-sideload"，一句话无法与 `ls` 调和。

`updater/src/preflight.rs` 让失败时说明原因；此文件是另一半：默认路径根本不能是 unit 会隐藏的路径。**两边各自都无法检查**——脚本不读 unit，unit 不知道脚本——所以在这里检查，整个仓库可读。

## 关键常量

- `SCRIPT = "scripts/dev-push.sh"`
- `UNIT = "updater/systemd/updaterd.service"`
- `PRIVATE = ["/tmp/", "/var/tmp/"]` — PrivateTmp 隐藏的目录

## 辅助函数

- `code(text)` — 去掉 `#` 注释（解释某路径*不*被用的注释不应被读成"使用"——注释存在的原因正是此文件要按住的 bug）
- `directive(unit, key)` — unit 文件的 `KEY=VALUE`，最后赋值生效（如 systemd）
- `privatises_tmp()` — PrivateTmp 是否为 yes/true/on/1

## 三个测试

1. **`the_sideload_path_is_not_one_private_tmp_hides`** — 若 unit 设 PrivateTmp=yes，脚本不得命名 `/tmp/` 或 `/var/tmp/`（否则 updaterd 读到自己的私有副本，什么都没有）
2. **`a_home_sideload_path_requires_no_protect_home`** — 若脚本用 `$HOME/duck-sideload`，unit 不得设 `ProtectHome`（同样的失败模式：ProtectHome 会隐藏家目录）
3. **`the_unit_still_privatises_tmp_or_this_file_is_moot`** — 守卫断言：unit 仍设 PrivateTmp=yes，或脚本仍从家目录 sideload。若两者都不再成立，上面两个条件测试都不再适用——必须删除本文件或恢复一半（防止条件测试静默退化）

## 失败模式

条件测试（前两个）停止适用时与通过无法区分——这就是第三个测试存在的原因。

## 关键摘要

sideload.rs 保证 `dev-push.sh` 的 sideload 路径不被 updaterd unit 的 `PrivateTmp=yes`（隐藏 /tmp、/var/tmp）或 `ProtectHome`（隐藏家目录）遮蔽。第三个测试是守卫：若 PrivateTmp 和家目录 sideload 都不再成立，前两个条件测试失效，必须删文件或恢复一半。

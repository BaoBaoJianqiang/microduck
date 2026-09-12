# 团队开发密钥

`team.dev.pub`——CI 给分支构建签名所用密钥的公钥半。一块板信任它之后，`robotctl update apply --ref <branch>` 才能用。

**刻意不放 [`../trusted_keys/`](../trusted_keys/)。** 那个目录会被 `scripts/install.sh` 复制到*每一台*机器人上，而信任这把密钥的机器人会安装团队里任何人构建的任何东西。它放在这里——没有任何东西默认安装它；`provision-board.sh` 负责把它送过去，`--no-dev-key` 则拒收。

**提交一把公钥不会泄漏任何东西。** 签署构建需要的是私钥半，它放在 `~/.duck-keys` 且从不离开。一块板子要接受 dev 构建，仍然需要两件各自独立的事同时成立——密钥在那块板的 `trusted_keys_dir` 里，*且*那块板的 `updater.toml` 里 `allow_dev_keys = true`。客户机器人上两者都关着，本文件的存在不改变其中任何一个。

它以前完全不进仓库。那并没有比上面两个开关多保护什么，却让每个新开发者都得为要一个文件多跑一趟。

如果私钥半丢失、需要重新生成——此后每块现有的开发板都要手工装上新公钥半，因为一块板只信任已经在其 `trusted_keys_dir` 里的东西：

```bash
minisign -R -s ~/.duck-keys/team.dev.key -p team.dev.pub
```

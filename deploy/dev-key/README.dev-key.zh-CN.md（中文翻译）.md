# 团队开发密钥

`team.dev.pub`——CI 用于签名分支构建的密钥的公钥部分，因此
`robotctl update apply --ref <branch>` 在信任它的板卡上可以工作。

**刻意不在 [`../trusted_keys/`](../trusted_keys/) 中。** 该目录由 `scripts/install.sh` 复制到
*每台*机器人上，而信任此密钥的机器人会安装团队中任何人构建的任何东西。此密钥改为放在这里，默认没有任何东西安装它——
`provision-board.sh` 会发送它，而 `--no-dev-key` 会拒绝。

**提交公钥不会泄露任何东西。** 签名构建需要私钥部分，它存放在
`~/.duck-keys` 中且永不离开。一块板卡仍然只有在两个独立条件同时成立时才接受 dev 构建——密钥在该板卡的 `trusted_keys_dir` 中，*并且*
其 `updater.toml` 中 `allow_dev_keys = true`。customer robot（客户机器人）上两者都关闭，且此文件的存在不会改变任何一个。

它以前完全不放在仓库中。那并没有保护任何上述两个标志尚未保护的东西，却让每个新开发者都要多跑一趟找人要文件。

如果私钥部分丢失，重新生成它——届时每台现有 dev board（开发板卡）都需要手工安装新的公钥部分，因为板卡只信任已存在于其
`trusted_keys_dir` 中的内容：

```bash
minisign -R -s ~/.duck-keys/team.dev.key -p team.dev.pub
```
#（注：内容由AI生成）

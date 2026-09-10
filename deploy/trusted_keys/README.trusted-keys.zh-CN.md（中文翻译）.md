# 受信任公钥

机器人的信任锚。`scripts/install.sh` 把这些复制到
`/etc/robot/trusted_keys/`，即 [`../updater.toml`](../updater.toml) 中的 `trusted_keys_dir`。
如果制品的签名能通过此处**任何一个**密钥的验证，该制品即可接受。

| | |
|---|---|
| `release-1.pub` | 当前签名所有发布和晋升（promotion） |
| `release-2.pub` | 如果 CI 或 `release-1` 被攻破，第一个轮换目标 |
| `release-3.pub` | 最后手段；其私钥部分永远不应接触联网机器 |

**三个都从第一个镜像起出厂，这正是全部意义所在。** 机器人只能对照烧录进自身的密钥集合进行验证，
所以一台只携带一个密钥、而该密钥私钥后来丢失或泄露的机器人，无法通过 over the air（空中更新）方式获得替换密钥——必须手工重刷（re-flashed by hand）。
现在附带备用密钥是免费的；事后加装则不可能。保管方式见
[`../../docs/project/ci-setup.md`](../../docs/project/ci-setup.md)。

**公钥不是秘密。** 提交它们是正确且有意的：它们正是必须公开、以便任何人验证我们签名的东西。
私钥部分存放在 `~/.duck-keys` 和密码管理器中，且只有 `release-1` 的私钥在 CI 中。

**`team.dev.pub` 刻意不在此目录中。** 信任开发密钥的机器人会安装团队中任何人构建的任何东西。
它只应存在于开发板的 `trusted_keys_dir` 中，与该板本地 `updater.toml` 中的 `allow_dev_keys = true` 并列——
永远不在这里，因为这里的内容每台机器人都会拿到。它提交在
[`../dev-key/`](../dev-key/)，默认没有任何东西安装它。

## 添加轮换密钥

生成它，把 `.pub` 放到这里，它会在机器人下次安装时到达。注意这种不对称性：一个*新*密钥只在添加之后
被镜像的机器人上才受信任，所以轮换保护的是未来的机群（going forward），无法拯救已在现场（already in the field）的机器人。
这就是备用密钥预先存在的原因。

```bash
cargo xtask keygen --kind release --name release-4 --out ~/.duck-keys
```
#（注：内容由AI生成）

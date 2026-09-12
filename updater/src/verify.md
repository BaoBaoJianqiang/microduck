# verify.rs 文件解析

## 文件位置

`d:\microduck\updater\src\verify.rs`

## 核心不变量

**无签名字节绝不提取到活路径或执行。** 验证顺序：manifest 签名 → 工件哈希 → 工件签名；提取仅在三者都通过后发生。

信任锚是磁盘上的 **minisign 公钥集合**（非单一内置密钥），使密钥丢失或泄露可应对。依赖 `minisign-verify`（零依赖、**仅验证**）而非完整 `minisign` crate——本进程无需签名能力，不应链接签名代码。

## `KeyRing`

- `load(dir, allow_dev_keys)` — 加载目录下所有 `*.pub`；**空 keyring 是错误**（不是空 allow-list）。文件名以 `.dev.pub` 结尾的密钥仅在 `allow_dev_keys=true` 时可用。
- `verify_bytes(data, signature)` — 内存字节分离签名验证（manifest）
- `verify_file(path, signature)` — **流式**文件验证（工件，不整个读入内存）
- 返回匹配的 `TrustedKey`，以便日志记录哪个密钥准入（发现仍依赖应退役的密钥）

`TrustedKey::Debug` 手写，绝不打印密钥材料，仅 id。

## `verify_sha256(path, expected_hex)`

流式 SHA-256，与 manifest hex 比较（不区分大小写）。仅完整性，非真实性（真实性来自签名）。

## `extract_artifact(archive, dest, limits)`

提取 `.tar.zst` 到 dest。**仅在签名+哈希都通过后调用。**

- 路径穿越安全来自 `tar::unpack_in`（拒绝绝对路径与逃逸目标的条目）；逃逸返回 `Verification` 错误（签名已通过，恶意条目意味着我们的密钥签了它，必须大声暴露）
- 额外限制：总未压缩大小（`max_uncompressed_bytes`，默认 2 GiB）与条目数（`max_entries`，默认 50000），防 zip bomb
- `set_preserve_permissions(true)` — 钩子与二进制需要 exec 位
- 超限返回 `ArchiveTooLarge`（非 Verification——非篡改）

## 测试

- `valid_signature_verifies` / `tampered_data_is_rejected` / `wrong_key_is_rejected`
- `dev_key_is_gated` — dev 密钥生产环境拒绝
- `file_signature_verifies_and_detects_tampering` — 流式文件验证
- `empty_keyring_is_an_error` — 空信任锚致命
- `sha256_detects_change` — 哈希大小写不敏感
- `refuses_path_traversal` / `refuses_absolute_path` — 路径穿越防护
- `refuses_too_many_entries` — 条目数限制
- `preserves_executable_bit` — exec 位保留

## 关键摘要

verify.rs 是信任与完整性边界：minisign 公钥集合（空即致命）+ 流式 SHA-256 + 流式签名验证；提取在三重验证后进行，含 tar 路径穿越防护与大小/条目上限防 zip bomb；依赖仅验证的 `minisign-verify` 避免链接签名能力。

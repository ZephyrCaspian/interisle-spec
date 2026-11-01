# 安全模型

> **状态：协议 v1。** 本文描述当前唯一的运行路径。catalog/mapping、解释报告、能力凭证和持有证明不属于协议 v1，也没有运行时路由；见[架构](02-architecture.md)。

## 威胁模型

| 威胁 | 场景 | 主要防线 |
|---|---|---|
| 双花 | 同一张凭证或同一个实例在同一个 claim domain 内的两个服同时兑换 | coordinator 的凭证 CAS + `(claim_domain, instance_id)` head/预留 + 目标 isle 隔离区 + TTL |
| 分叉 | 同一实例的两张不同凭证都试图推进 head | head 只接受当前 head 的后继；genesis 只建一次；abort 不后退 |
| 伪造凭证 | 玩家或服务器编造所有权 | 签发方 Ed25519 签名 + 社区信任名单 + 公钥 pinning |
| 重放 | 旧凭证被重复提交 | `credential_id` 一次花费；已花凭证返回 `CREDENTIAL_SPENT` |
| 恶意签发方 | A 签发虚高属性或不存在的东西 | 社区只信任 isle；目标 isle 可选择不接受某签发方。v1 没有 catalog 经济平衡或保真度报告 |
| 恶意 coordinator | 回滚 claim 状态、审查、放行双花 | 社区选择与运营问责；成员可换社区；协议不提供透明日志 |
| 恶意源 isle | A 签了凭证但没删除本地物品 | **协议无法阻止**；源 isle 是受信的，只能靠运营问责 |
| 密钥泄露 | 服务器或玩家私钥被盗 | 口令加密的身份钥、owner-only 文件/ACL、加密备份、重新签发、从信任名单移除签发方；无逐凭证撤销 |
| 越权提取 | 客户端尝试提取世界物品、他人物品或非当前角色物品 | isle 校验 owner 和 `(owner, character)` 的打开会话背包；不在背包返回 `NOT_IN_BACKPACK`；控制面需要 bearer token |
| 令牌泄露 | token 出现在 argv、环境变量、日志或被其他用户读取 | 只从 owner-only 文件读取；Unix 检查 `mode & 0o077`，Windows 检查所有者、受保护 DACL 和 ACE；`curl -H @file` 避免 argv |
| 语义膨胀 | A 的高数值物品冲垮 B 经济 | v1 没有自动缩放或拒绝规则；B 通过社区信任名单和本地适配器自行决定是否接受。这是当前未实现项 |
| 中间人/流量分析 | 公网传输被截获 | TLS + 对象签名；没有 relay，visitor 不对外提供服务 |

## 密码学基线

- **签名**：Ed25519；账号、isle、coordinator 和签发方都用 `did:key` 标识。
- **规范化**：JSON + JCS（RFC 8785）。签名覆盖 `domain || 0x00 || JCS(body without sig)`。
- **哈希**：SHA-256。带 ID 的对象满足 `id = sha256(JCS(body without own id and sig))`；签名因此覆盖 ID。
- **原始字节验证**：验签前拒绝重复键、浮点和超出安全范围的整数，避免不同 JSON 解析器产生不同对象。
- **域分隔**：物品凭证使用 `interisle/v1/credential`，presentation manifest 使用 `interisle/v1/presentation-manifest`，信任名单和节点 manifest 分别使用 `interisle/v1/trust-list` 和 `interisle/v1/manifest`。不同 domain 的对象不能互相重放。
- **数值**：整数或定点数，禁止浮点。
- **传输**：节点之间用 HTTPS；适配器 sidecar 默认用 stdio JSON-RPC（无端口、无 token），兼容模式只绑 loopback TCP + 每实例 bearer token。

有效期：

- coordinator 预留默认由 isle 设为 120000 ms，协议上限 600000 ms；
- 预留 TTL 到期后惰性释放，目标 isle 的隔离物品回滚；
- 物品凭证本身没有过期时间，只在目标服接受规则变化或签发方被移出信任名单时失效。

## 本地密钥与令牌

物品凭证的安全性最终落在账号私钥和节点控制面 token 上，因此本地存储按敏感数据处理：

- **身份钥**使用 `interisle-key/v1` 信封：Argon2id（默认 64 MiB、3 次迭代、1 并行）从口令派生密钥加密密钥，XChaCha20-Poly1305 包裹随机数据密钥，数据密钥再加密种子和 profile id。口令至少 12 个字符，且只从交互提示读取，不接受 argv、环境变量、配置文件或 URL。
- 旧的明文 32 字节种子可以用 `key migrate` 原地迁移；迁移先写入并验证加密文件，成功后才移除明文。`key backup` / `key restore` 使用同样的 KDF + AEAD 加密备份，恢复前要求显式确认覆盖。
- **控制面 token** 只从 `*_token_file` / `--token-file` 等文件引用读取。文件必须是普通文件、非空、不超过 4096 字节；Unix 上要求 `mode & 0o077 == 0`；Windows 上要求所有者是当前运行账户、DACL 存在且 `SE_DACL_PROTECTED`，每个 ACE 都是非继承的 `ACCESS_ALLOWED` 且只授权当前账户。失败返回稳定错误码（`KEY_INSECURE_PERMISSIONS`、`KEY_EMPTY_TOKEN`、`KEY_TOKEN_TOO_LARGE`），不会降级读取。
- token 不出现在命令行参数、环境变量、日志或 URL 中；curl 示例使用 owner-only header 文件（`curl -H @file`）。keyring 集成和完整短时 token 尚未实现，见 ADR 0013。

## 防双花与防分叉

防双花靠两个唯一性约束，而不是“尽量检查”。以下保证只在同一个 coordinator / **claim domain** 内成立：

1. coordinator 对每张 `credential_id` 维护 `CLAIMABLE` / `RESERVED` / `SPENT` 状态。
2. `claim` 是原子的 CAS：只有 `CLAIMABLE`（或已过期的 `RESERVED`）能变成某个目标的 `RESERVED`。
3. 对带 `instance` 的凭证，coordinator 同时检查 `(claim_domain, instance_id)` 的当前 head：
   - genesis 只能在该实例还没有 head 时 claim；
   - extract 的 parent 必须是当前 head 的凭证，owner 相同，`sequence` 正好是 head 加一；
   - 同一实例已有另一个未过期预留时返回 `INSTANCE_BUSY`。
4. 两个并发 claim 只有一个成功；另一个收到 `CONFLICT`、`INSTANCE_BUSY` 或 `INSTANCE_CONFLICT`。
5. 目标 isle 在 commit 之前把原生物品放在**隔离区**，不可使用。
6. commit 成功后 coordinator 原子地把凭证变 `SPENT` 并创建/推进 head，目标 isle 才把物品转正；visitor 随后删除本地凭证。
7. 如果目标崩溃，TTL 释放预留；重启后按 credential/reservation 幂等补 commit 或回滚。abort 不会让 head 后退。
8. commit 响应丢失时重试安全；不会产生第二次花费或第二个 head。

**没有本地信任降级。** coordinator 不可达时，兑换失败关闭；玩家看到的是“稍后重试”，不是“物品已到账但可能重复”。提取是例外：它不需要 coordinator，由源 isle 自己完成，但只对打开会话背包中的本地物品生效。

coordinator 是单写者，所有 claim 操作串行化，SQLite 约束保证持久性。coordinator 重启后状态从数据库恢复；恢复的数据库就是恢复 claim 状态和 head，不是重放事件。

跨 claim domain 没有全局唯一性，但 v1 用签名里的 `claim_domain` 阻止同一张凭证被两个中心直接 claim：目标 isle 中心不匹配时返回 `WRONG_CLAIM_DOMAIN`，coordinator 也只接受自己域内的凭证。v1 因此仍是单域协议，不能把外部 issuer 加入信任名单当作跨社区通道。跨社区物品移动需要源域消费/回执、目标域验证/一次性消费和超时处理的转移协议；见 ADR 0017。

## 权威与边界

- **isle 是本地物品和会话背包的权威**：它在自己的数据库里记录 owner、角色、descriptor、native_ref、instance、sequence、进入本地世界时的凭证 ID，以及物品属于哪个 `(owner, character)` 的会话背包。
- **仓库是凭证的持有者**：visitor 只保存凭证和 `held` / `pending`，不知道物品在世界里的位置，也不是会话背包的第二份副本。visitor 的账号密钥不是 item issuer，不能签发凭证。
- **coordinator 是 claim 状态和 head 的权威**：它不知道世界状态、会话或背包，也不签署“物品存在”证明。
- **源 isle 是受信的**：A 在提取时删除本地物品和背包项并签出 C1；B 只能验证 C1 的签名和 head 链，无法验证 A 真的删了物品。协议只防同一张凭证/实例在同一个 claim domain 内被花两次，不防服务器复制或丢失物品。
- **关闭/崩溃不改变所有权边界**：`close` 只改会话状态，强退/掉线也不会自动提取或回收；物品留在该 isle，玩家必须回来从同一会话背包提取。v1 没有出服授权、outbox 或自动恢复。
- **没有全局物品 ID**：本地 `item_id` 只在本 isle 有意义；跨服身份由凭证的 `instance_id`、签名和 lineage 表达。

## coordinator 的可信度与局限

coordinator 是运营者，不是资产所有者。它：

- 能验证凭证签名、登记 claim、拒绝服务、审查某个玩家或 isle；
- 能发布新的信任名单，决定哪些 isle 被社区接受为物品签发方；
- **不能伪造**源 isle 签名的物品凭证；
- **不能阻止**自己作恶：拥有数据库的运营者可以恢复旧快照、放行已花凭证或隐瞒状态。当前没有只追加事件日志、Merkle checkpoint 或透明日志来检测这种回滚。
- 一旦社区中心不可信，成员可以停止使用它，或迁移到另一个社区中心；但已经发生的双花无法由密码学自动修复。

这是有意的取舍：用“一个可问责的社区中心 + 本地世界权威”换取简单、离线提取和低运营成本。高价值场景需要额外的审计、第三方见证或多签运营，这些不在 MVP 内。

## 信任名单与签发方

- 社区信任名单由 coordinator 签名，列出可信的**物品签发方**，通常就是 isle。
- 验证时逻辑 `issuer` 和实际 `key_id` 都必须出现在名单；只匹配其中一个不够。
- isle 可以用 `--center` pin 中心 DID，拒绝其他中心签名的名单；也可以用 `--trust-item-allow/deny` 做本地覆盖，deny 优先。
- 名单是签发方级别，不是逐凭证撤销。要停止信任某个签发方：重新发布名单，或在 isle 上 deny。
- 物品凭证没有过期和逐凭证撤销；已经签出的凭证在目标服接受规则不变时继续有效。
- 能力签发方和能力凭证不属于协议 v1；信任名单只列物品签发方。

## claim domain 与跨社区

- **v1 是单域协议。** 一个逻辑 isle 只 pin 一个社区中心，只有一份信任名单和一份本地物品状态；这个中心就是它的 home claim domain。每张凭证的 `claim_domain` 必须等于它。
- 信任名单只决定 issuer 是否受信，不是 claim 状态。把外部 issuer 加进名单，不会让目标中心知道源中心是否已经花过这张凭证。
- isle 在向 coordinator 注册凭证前检查 `claim_domain`；不匹配直接返回 `WRONG_CLAIM_DOMAIN`。这关闭了“两个独立中心各 claim 一次同一张凭证”的直接路径，但不提供跨社区转移。
- 部署规则仍是：物品信任名单只列本域签发方；不接受外部物品凭证。`--trust-item-allow` 只改变 issuer 信任，不创建域间 claim 同步。
- 跨社区物品移动必须走显式转移：源域消费/锁定凭证并签回执，目标域验证并一次性消费回执，再落地目标域物品；需要 ack/timeout、重放保护和桥的信任假设。MVP 不实现，见 ADR 0017。
- 一台物理主机要服务多个社区，运行多个隔离的逻辑 isle，每个归属一个 claim domain；不要用一个 isle 同时接多个独立中心。

## 身份与角色

- 账号 DID 是仓库和物品所有权的根；物品凭证归账号所有。
- 角色是账号下的稳定标签 `character_id`，不是新密钥、新仓库或新 DID。同一时间只有一个活动角色，切换在客户端完成。
- v1 的提取和兑换由 visitor 的控制面 token 保护，isle 校验 owner 和打开会话背包；凭证本身不携带角色签名。角色隔离是会话级保证，不是跨节点的密码学证明。完整的角色绑定/短期 scoped token 仍未实现。

## 非目标

- **不提供匿名性**：节点和玩家身份是公钥，不是匿名凭证；coordinator 能看到凭证、owner、目标和角色。
- **不做 DRM**：协议证明凭证的所有权和一次花费，不阻止游戏内复制模型或私服改数据。
- **不做通用共识**：不需要区块链；coordinator 是常规数据库。
- **不保证语义等价**：目标 isle 只按自己的适配器决定如何物化物品；v1 没有签名保真度报告。
- **不做在线联邦**：一个社区一个中心，v1 是单 claim domain；跨社区物品转移不在 MVP，直接信任外部 issuer 会失去跨中心双花防护。

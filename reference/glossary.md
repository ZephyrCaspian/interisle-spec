# 术语表

| 术语 | 英文 | 含义 |
|---|---|---|
| 岛 | isle | 一个独立的游戏服务器节点；拥有本地物品、会话/背包、世界状态和适配器 |
| 访客节点 | visitor | 玩家随身的最小服务器，保存可花费凭证、角色和入站 API，并代理 isle 会话；不保存物品，没有内嵌 isle |
| 家园 | Home | launcher 的本地非经济世界，由 `isl-home` 承载；可编辑、可好友联机，不签发、托管、兑换或提取凭证 |
| 世界包 | `.islworld` | 内容寻址的 ZIP，含 `world.json`、初始对象和渲染器；包本身不签名 |
| 渲染器 | renderer | 读取 Home HTTP API 的显示客户端；Godot 渲染器由独立 `renderer.godot` 插件携带导出的运行时；只拿短期会话令牌，可替换 |
| 家园邀请码 | Home invite code | 主机生成的一次性 `<invite_id>.<secret>`；数据库只保存 secret 的 SHA-256 |
| 家园会话令牌 | Home session token | 加入或重连后由 Home 返回的短期 bearer token；不是账号密钥，不能访问经济路由 |
| 社区中心 | coordinator | 记录凭证 claim 状态和 instance head、发布签名信任名单、托管目录；每个社区一个 |
| 适配器 | adapter | isle 与游戏引擎之间的桥；可以是进程内实现，或通过 `interisle.adapter/v1` 通信的 sidecar |
| 仓库 | warehouse | visitor 的本地凭证存储，按 `credential_id` 保存 `held`/`pending` 凭证；不保存世界内位置 |
| 背包 | backpack | isle 权威的会话内本地物品集合；只有背包里的本地物品能提取 |
| 会话 | session | isle 为 `(isle_id, owner, character_id)` 维护的打开/关闭状态；同一组合只有一个会话 |
| 会话打开 | session open | 创建或恢复会话；`import_native` 和兑换 commit 只会把物品加入已打开的背包 |
| 会话关闭 | session close | 只把会话标为 `CLOSED`，不提取物品；当前没有出服授权或 outbox |
| 账号 | account | 仓库根身份，用 `did:key` 表示；物品凭证归账号所有 |
| 角色 | character | 账号下的稳定标签 `character_id`；同一时间只有一个活动角色，不是新密钥 |
| 描述符 | descriptor | 与游戏引擎无关的物品描述：class、attributes、effects、transfer 等 |
| 实例 | instance | 一件真实物品的稳定身份；`instance_id = "iid:" + 64 hex`，凭证携带各游戏的原生快照 |
| 实例 head | instance head | coordinator 按 `(claim_domain, instance_id)` 记录的最新已提交凭证；genesis 只建一次，extract 只能基于当前 head 推进 |
| 原生快照 | native snapshot | 某个游戏对这件物品的完整原生数据，存在凭证的 `instance` 和 isle 本地记录中；目标适配器优先使用自己游戏的快照 |
| 表现引用 | presentation | 指向本地名称、图标、模型等表现资源的引用；资源不随物品转移，目标游戏不拥有源资产 |
| 物品凭证 | item credential | isle 签名的、不可变、可一次性花费的整件物品表示，带 `instance`、`claim_domain`、`sequence`、`parents` 和操作记录 |
| 仓库凭证 | warehouse credential | 同一张凭证在 visitor 仓库中的持有状态；仓库不签凭证，只保存签发方签名过的字节 |
| 本地物品 | local item | isle 世界里的物品记录，拥有 `item_id`、`native_ref`、owner、角色和 lineage；可能在世界里或会话背包中 |
| 原生引用 | native_ref | 游戏引擎自己的物品实例标识；只对当前 isle 有意义 |
| 物品 ID | item_id | isle 为本地物品生成的标识；不是全局物品 ID |
| 谱系 | lineage | 凭证的 `sequence`、`parents` 和 `operation`：每次提取递增并指向上一张凭证；genesis 的 `sequence = 0` |
| 提取 | extract | 源 isle 验证 owner 和背包位置后删除**会话背包中的**本地物品并签出下一张凭证；不需要 coordinator |
| 兑换 | redemption | 目标 isle 验证凭证、向 coordinator 预留并物化本地物品；commit 时推进 instance head |
| 预留 | reservation / claim | coordinator 给某个目标 isle 的限时占位，防止同一凭证或同一实例被两个目标同时兑换 |
| 可认领 | CLAIMABLE | 凭证当前可被 claim 的状态 |
| 已预留 | RESERVED | 某个目标持有未过期预留；其他目标 claim 会得到 `CONFLICT` 或 `INSTANCE_BUSY` |
| 已花费 | SPENT | 已经 commit，永久不可再花 |
| 隔离区 | quarantine | 目标 isle 已实例化但尚未 commit 的物品；不可使用，可回滚 |
| 提交 | commit | 目标 isle 在 coordinator 花掉凭证并推进 head，然后把隔离物品转正 |
| 中止 | abort | 释放预留，回滚隔离物品，凭证回到 `CLAIMABLE`；head 不后退 |
| 目录 | directory | 节点自签名记录的发现索引；不背书身份 |
| 信任名单 | trust list | coordinator 签名的物品签发方名单；逻辑 issuer 和 key ID 都要匹配。只列 isle |
| claim domain | claim domain | 一个 coordinator 对凭证花费状态和 instance head 唯一负责的范围；当前一个社区一个域，社区中心 DID 是标识 |
| 归属域 | home claim domain | 一张凭证当前所属的 claim domain；每张凭证任一时刻只有一个归属域 |
| 跨社区转移 | cross-community transfer | 源域消费/锁定凭证并签回执、目标域验证并一次性消费后落地的协议；不在 MVP |
| 多租户 isle | multi-tenant isle | 一台物理主机运行多个隔离的逻辑 isle，每个 isle 有自己的密钥、数据库、端口和归属社区 |
| JCS | JSON Canonicalization Scheme | RFC 8785，签名前的 JSON 规范化 |
| did:key | — | 用公钥直接编码的 DID；账号、isle、coordinator 和签发方的身份表示 |
| TOFU | trust on first use | 首次接触时信任并 pin 公钥，之后变更需显式确认 |

历史术语：`relay`、`mini-isle`、`inbox`、`ability credential`、`catalog`、`mapping`、`fidelity report`、`possession proof`、`split/merge` 和 `CLAIM_PENDING` 属于早期草案或已删除组件，不在协议 v1 中；ADR 保留作历史记录。

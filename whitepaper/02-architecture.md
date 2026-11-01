# 架构

## 角色

```text
                    ┌──────────────────────────┐
                    │ coordinator（社区中心）    │
                    │ claim/instance head/信任/目录│
                    └────▲──────────────▲──────┘
                  claim/commit      注册/发现
                    ┌────┴────┐    ┌────┴─────┐
                    │ isle A  │    │ isle B   │   游戏服务器 + 适配器
                    └──▲────▲─┘    └─▲──────▲─┘
              本地物品 │    │凭证       │      │
                    ┌─┴────┐        ┌─┴──────┐
                    │ game │        │visitor │  玩家随身节点
                    │server│        │仓库+会话代理│
                    └──────┘        └────────┘
```

- **isle（岛节点）**：一个游戏服务器的节点。负责本地物品、会话和会话背包、extract/redeem、隔离区和恢复；不嵌入游戏进程。
- **visitor（访客节点）**：玩家随身的最小服务器。只保存可花费凭证、角色和入站 API，并代理 isle 的会话/背包；不保存物品，也没有 mini-isle。
- **coordinator（社区中心）**：记录每张便携凭证的 `CLAIMABLE` / `RESERVED` / `SPENT`，按 `(claim_domain, instance_id)` 维护 head 和预留，发布社区信任名单，托管内置目录。它没有事件日志，不按资产分片。
- **game server / adapter**：游戏自己的服务端逻辑，通过进程内适配器或独立 sidecar 与 isle 通信，拥有世界内的物品和位置。
- **launcher（Tauri 2 + Vue）**：玩家入口，启动并监管本机 visitor，管理 profile、插件和 Home；私钥不进入 launcher。

isle 之间不直接通信。跨服携带靠玩家仓库里的凭证：源 isle 在提取时签出新凭证并删除本地物品；目标 isle 在兑换时只联系 coordinator 做预留和 head 检查，不需要联系源 isle。A 在提取完成时就承认“物品已被带走”，之后不再跟踪它。会话和背包是 isle 的本地状态，coordinator 不感知。

非经济 Home 是独立进程 `isl-home`，由 launcher 作为世界插件启动；它不属于经济路径，不签发、托管、兑换或提取凭证。

## 分层

| 层 | 职责 | 关键约束 |
|---|---|---|
| L0 传输 | HTTPS + JSON（JCS 规范化） | 先 JSON；性能不够再考虑其他编码 |
| L1 身份/密码 | Ed25519、did:key、TOFU/pinning、密钥轮换 | 节点 ID = 公钥指纹 |
| L2 契约 | descriptor / instance / credential / trust-list / manifest | schema 是唯一真源 |
| L3 状态机 | 提取、预留、提交、回滚、TTL、instance head | 幂等 + 一次花费 + 失败关闭 |
| L4 运行时 | isle / visitor / coordinator | 单二进制，SQLite 内嵌 |
| L5 适配 | 进程内参考适配器或 `interisle.adapter/v1` sidecar | 游戏差异全部在适配器侧 |

## 物品在哪里

协议不维护一个全局的“物品当前位置”。只有两种表示，isle 本地物品又分为背包内和背包外：

```text
仓库里的凭证 C1                      isle 的本地物品
   owner = account DID      或者      owner = account DID
   descriptor + instance（原生快照）    descriptor + native_ref + instance
   sequence / parents / operation     item_id（仅本 isle 有效）
                                      + 是否在 (owner, character) 的打开会话背包中
```

- 物品在 A 的世界里时，A 的本地记录存在，仓库里没有对应的可花费凭证。本地物品可能在世界位置、容器里，也可能在会话背包中。
- 只有打开会话背包中的本地物品能 `extract`；其他位置返回 `NOT_IN_BACKPACK`，留在 isle。
- 从 A 提取后，A 删除本地记录和背包项并签出 C1；仓库里有 C1，A 的世界里没有这件物品。
- 在 B 兑换成功后，B 创建本地物品并加入 B 的打开会话背包，C1 在 coordinator 变成 `SPENT`，visitor 删除 C1；仓库里没有它，B 的背包里有它。
- `close` 只把会话标为 `CLOSED`，不改变物品位置；强退/掉线/游戏崩溃后物品仍在 isle，重新打开同一会话会恢复背包，但没有自动回收。
- 一张凭证仍然是一个可花费输出。v1 的运行路径没有 split/merge；需要部分数量时必须等后续协议，现有 `splittable` 字段不改变单件实例语义。

## 实例身份

v1 的每件不可堆叠物品有一个稳定 `instance_id = "iid:" + 64 hex`，在首次进入仓库时确定，之后跨 isle、跨游戏保持不变。凭证携带：

- `descriptor`：与引擎无关的 class、attributes、effects 和 transfer 信息。
- `instance`：来源游戏的 origin（isle、game、native_ref）、跨游戏保留的原生快照，以及可选的 `presentation` 引用。
- `sequence` / `parents` / `operation`：每次提取加一，`parents` 指向上一张凭证；genesis 的 `sequence = 0`。
- `claim_domain`：签发时绑定的 community DID。

目标适配器优先使用自己游戏的原生快照；没有该游戏的快照时才从 descriptor 创建。目标适配器只更新自己游戏的原生命名空间，其他游戏的数据原样保留。这就是“从 A 带一把剑到 B，B 仍能保留 A 的细节”的实现基础。

## 凭证与 claim 状态机

一张物品凭证是不可变的、可花费的输出。coordinator 维护它的 claim 状态和实例 head：

```text
CLAIMABLE ──claim(B)──► RESERVED ──commit(B)──► SPENT
    ▲                       │
    └──── abort / TTL ──────┘
```

- `CLAIMABLE`：没有被任何目标预留，可以被 claim。
- `RESERVED`：B 持有未过期的 reservation；其他 isle claim 会得到 `CONFLICT` 或 `INSTANCE_BUSY`。
- `SPENT`：已经 commit，永久不可再花。
- abort 或 TTL 到期回到 `CLAIMABLE`；TTL 是惰性处理的。
- 同一个凭证 + 同一目标的重试是幂等的；重复 commit 不会产生第二次花费。
- coordinator 不写世界事件，也不签“资产回执”；它只持久化 claim 状态和 head。
- head 只在 commit 时创建/推进，abort 不会让它后退。genesis 只能在该实例还没有 head 时创建；extract 必须指向当前 head、同一 owner，且 sequence 正好加一。

## 一次跨服携带

1. 玩家在 A 打开 `(owner, character)` 会话，并把要带出的物品加入该会话背包。已经在世界里的物品需要重新 `import` 一次才会加入已打开的背包；`open` 不会追溯已有本地物品。
2. 玩家通过本机 visitor 发起提取；visitor 用控制 token 调用 A 的 `POST /v1/items/extract`。
3. A 验证物品存在、owner 匹配且位于打开会话背包中，删除本地物品和背包项，签出 C1。**A 不联系 coordinator。**
4. visitor 把 C1 放进仓库；仓库只保存凭证和 `held` / `pending` 状态。
5. 玩家到 B，visitor 先确保 B 有同一 `(owner, character)` 的打开会话，再调用 B 的 `POST /v1/redeem`，请求体是 `{credential, player, character_id}`。
6. B 验签、验证 `claim_domain` 和 issuer 信任，然后向 coordinator claim；coordinator 同时检查凭证状态和 instance head。拿不到预留或 head 冲突就不实例化物品。
7. B 把实例化后的原生物品放进**隔离区**，返回 pending；visitor 把凭证标为 `pending`。
8. 玩家确认后 commit：B 在 coordinator 把预留 CAS 成 `SPENT` 并推进 head，成功后把隔离物品转正并加入 B 的打开会话背包；visitor 删除本地凭证。
9. 玩家从 B 带出时，只能从 B 的打开会话背包提取；B 删除本地物品并签 C2（`sequence = 1`，`parents = [C1]`，同一个 `claim_domain`）。回到 A 时重复兑换，形成 round-trip。

这些步骤的失败边界：

- 提取在 coordinator 离线时仍能完成；源 isle 是唯一执行者。物品不在打开会话背包中时返回 `NOT_IN_BACKPACK`。
- 兑换必须经过 coordinator；coordinator 不可达时 begin 失败，物品不会“先给了再说”。
- commit 后如果响应丢失，重试是幂等的；不会多花一次。
- TTL 到期后预留释放，隔离物品回滚。
- `close` 不会自动提取；强退/掉线/崩溃后背包物品留在该 isle。当前没有自动回收，也没有跨 isle 找回。

## 跨游戏表现

v1 不实现 catalog/mapping/解释报告/能力凭证。目标 isle 的适配器按以下顺序物化物品：

1. 凭证里有本游戏的 `instance` 原生快照：直接使用，保留逐项数值和自定义字段。
2. 只有其他游戏的快照：从 `descriptor` 按本游戏规则创建，并把 origin 和其他游戏的快照留在凭证/本地记录里。
3. `presentation` 引用用于本地表现（名称、图标、模型）；它只是引用，目标游戏不拥有源资产。

跨游戏不是忠实复制，也不承诺能力等价；当前承诺的是**身份、origin、逐游戏原生数据与本地表现可追溯**。原能力执行、效果解释和保真度报告不属于协议 v1，也没有运行时路由。

## 信任边界

- coordinator 可信的部分：claim 的唯一性、instance head 的单调推进、信任名单的签名、目录记录的签名校验；它不能伪造源 isle 的物品凭证。
- coordinator 不可信/不负责的部分：物品语义、世界状态、准入判断。
- 源 isle 是受信的：它签出的凭证代表它确实删除了本地物品；协议不能阻止恶意 isle 复制物品。
- 玩家信任自己的 visitor 节点；目标 isle 通过 owner 字段和 isle token 控制面验证操作。
- 只有 isle 签发物品凭证；visitor 的账号密钥是持有者，不是 item issuer，社区信任名单只列 isle。
- isle 可以 pin 社区中心 DID（`--center`）；信任是签发方级别的，逻辑 issuer 和签名 key ID 都要匹配。
- 没有 relay；visitor 只服务本机 launcher 和游戏进程。

## claim domain 与多社区

- 一个 coordinator 负责一个 **claim domain**：该域内凭证的 `CLAIMABLE` / `RESERVED` / `SPENT` 状态和 instance head 由它唯一裁决。当前一个社区一个中心，因此社区中心 DID 就是域标识。
- 每张物品凭证在任一时刻只有一个 home claim domain。v1 把 `claim_domain` 写进签名载荷；目标 isle 必须固定同一个中心，否则在联系 coordinator 前就返回 `WRONG_CLAIM_DOMAIN`。
- 信任名单只回答“本域接受哪些签发方”，不回答“这张凭证有没有在别的中心花过”。即便同一 issuer 被两个社区信任，凭证的 `claim_domain` 也只能指向一个中心；跨域凭证不能直接 claim。
- 跨社区物品移动需要显式的转移协议：源域消费/锁定凭证并签回执，目标域验证并一次性消费回执，再落地物品。该协议不在 MVP，见 ADR 0017。
- 一个物理主机要服务多个社区，运行多个隔离的逻辑 isle（独立密钥、数据库、端口和 `--center`），每个 isle 归属一个 claim domain；不要用一个 isle 同时接多个独立中心。
- 账号 DID 可以跨社区；身份连续不代表物品的 claim 状态连续。

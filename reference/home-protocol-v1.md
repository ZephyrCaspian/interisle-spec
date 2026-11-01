# Home 协议参考（`interisle.world/v1`）


Home 是一个主机权威、非经济的可编辑世界服务。它只保存世界对象、成员、邀请和审计，不保存凭证、仓库、账号密钥或任何签发材料。本文描述线格式和 HTTP 路由；渲染器实现只能依赖这里和共享 `isl-world` crate 的校验规则。

## 1. 两个格式

| 名称 | 值 | 用途 |
|---|---|---|
| 世界协议 | `interisle.world/v1` | HTTP 响应、加入证明、快照 |
| 包格式 | `interisle-world/1` | `.islworld` ZIP 内的 `world.json` |

`.islworld` 是 ZIP；根目录必须有 `world.json`。包按 manifest 自哈希寻址，不签名。加入前，访客必须：

1. 从 `/v1/health` 取得 `content_hash`；
2. 从 `/v1/package` 下载 ZIP；
3. 校验 manifest 自哈希、每个文件的 SHA-256、渲染器条目和路径；
4. 只有校验通过后才签署加入证明。

## 2. 时间、坐标和身份

- 时间戳是 Unix 毫秒（`u64`）。
- 坐标是整数网格：`{"x": i32, "y": i32, "z": i32, "rotation": 0..=3}`。坐标绝对值不超过 1,000,000；不使用浮点数。
- 账号和 Home 都是 `did:key:` Ed25519 DID。
- 角色：`owner`、`builder`、`visitor`。`owner` 和 `builder` 可以编辑；只有 `owner` 可以管理成员、邀请、备份和回滚。
- 世界对象 ID 是 1–64 字节的 ASCII 字母、数字、`-`、`_`；服务器可分配 `obj_<hex>`。

## 3. 签名对象

### 3.1 加入挑战

`GET /v1/challenge` 返回由 Home 节点密钥签名的挑战：

```json
{
  "protocol": "interisle.world/v1",
  "home_did": "did:key:z...",
  "world_id": "home.default",
  "world_name": "Home",
  "content_hash": "sha256:...",
  "challenge_id": "chl_...",
  "nonce": "...",
  "issued_at": 1760000000000,
  "expires_at": 1760000120000,
  "sig": "..."
}
```

签名域：`interisle.world/v1/challenge`。签名字段 `sig` 不参与规范化。挑战生命周期不得超过 2 分钟，且只能使用一次。

### 3.2 加入证明

访客用账号密钥签署 `HomeJoinProof`，随 `POST /v1/join` 发送：

```json
{
  "challenge": { "…": "原样返回" },
  "proof": {
    "v": 1,
    "kind": "interisle.home.join",
    "home_did": "did:key:z...",
    "invite_id": "inv_...",
    "account_did": "did:key:z...",
    "display_name": "Guest",
    "challenge_id": "chl_...",
    "nonce": "...",
    "issued_at": 1760000000000,
    "expires_at": 1760000060000,
    "sig": "..."
  },
  "invite_code": "inv_....<secret>"
}
```

签名域：`interisle.world/v1/join-proof`。`sig` 不参与规范化。证明有效期最长 5 分钟。首次加入必须带邀请码；已激活成员重连时 `invite_id` 为 `"rejoin"` 且省略 `invite_code`。

成功响应：

```json
{
  "protocol": "interisle.world/v1",
  "session_token": "<短期 bearer token>",
  "role": "builder",
  "expires_at": 1760003600000,
  "snapshot": { "…": "WorldSnapshot" }
}
```

`session_token` 只返回一次，Home 只保存其摘要。响应绝不包含 owner-control token。

## 4. HTTP 路由

### 4.1 公开路由

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/v1/health` | 世界 ID、名称、Home DID、owner DID、内容哈希、修订、`joinable` |
| GET | `/v1/challenge` | 一次性签名挑战 |
| POST | `/v1/join` | 验证证明、邀请码和内容哈希，返回会话 token |
| GET | `/v1/package` | 返回原始 `.islworld` ZIP |
| GET | `/` 或 `/world` | 302 到 Web 渲染器入口 |
| GET | `/world/*` | 渲染器静态资源；HTML 带 CSP |

### 4.2 需要 bearer token

| 方法 | 路径 | 最低角色 | 说明 |
|---|---|---|---|
| GET | `/v1/world` | visitor | 当前世界快照 |
| GET | `/v1/events?since=R&timeout_ms=N` | visitor | 长轮询修订通知，timeout ≤ 30,000 ms |
| POST | `/v1/objects` | builder | 创建或更新对象；可带 `expected_revision` |
| DELETE | `/v1/objects/{id}` | builder | 删除对象 |
| POST | `/v1/presence` | visitor | 心跳和当前位置 |
| POST | `/v1/leave` | visitor | 结束会话 |
| GET | `/v1/members` | visitor | 成员列表（在线状态） |
| GET | `/v1/sessions` | visitor | 未过期会话 |
| POST | `/v1/admin/renderer-session` | owner | 为渲染器签发短期会话 token |
| POST | `/v1/admin/invites` | owner | 创建一次性邀请码 |
| GET | `/v1/admin/invites` | owner | 列出邀请 |
| POST | `/v1/admin/invites/{id}/revoke` | owner | 撤销邀请 |
| POST | `/v1/admin/members/{did}/role` | owner | 修改角色 |
| POST | `/v1/admin/members/{did}/kick` | owner | 踢出并撤销会话 |
| POST | `/v1/admin/members/{did}/ban` | owner | 封禁 |
| POST | `/v1/admin/members/{did}/unban` | owner | 解封 |
| POST | `/v1/admin/backup` | owner | 生成一致性备份 |
| POST | `/v1/admin/rollback` | owner | 回滚到指定修订 |
| GET | `/v1/admin/audit?limit=N` | owner | 审计记录，默认 100 |

Home **没有** `/v1/issue`、`/v1/issuance`、`/v1/credentials`、`/v1/extract`、`/v1/claim`、`/v1/redeem`、`/v1/warehouse` 等经济路由；这些路径返回 404。

## 5. 世界快照

```json
{
  "protocol": "interisle.world/v1",
  "world_id": "home.default",
  "name": "Home",
  "home_did": "did:key:z...",
  "content_hash": "sha256:...",
  "revision": 12,
  "role": "builder",
  "actor": "did:key:z...",
  "objects": [
    {
      "id": "obj_...",
      "kind": "block:stone",
      "owner_did": "did:key:z...",
      "display_ref": "item:sword#display",
      "transform": {"x": 0, "y": 0, "z": 0, "rotation": 0},
      "properties": {},
      "revision": 12,
      "updated_at": 1760000000000
    }
  ],
  "presence": [
    {
      "account_did": "did:key:z...",
      "display_name": "Guest",
      "role": "builder",
      "transform": {"x": 2, "y": 0, "z": 1, "rotation": 0},
      "last_seen_ms": 1760000000000
    }
  ]
}
```

`display_ref` 是不透明显示引用，最多 256 个字符；不能包含凭证、bearer、private、seed 等字样。`properties` 必须是 JSON 对象，最大 4 KiB，嵌套不超过 8 层，并拒绝 `credential`、`token`、`secret`、`seed`、`private_key`、`signature`、`sig` 等字段。

## 6. 编辑与并发

```http
POST /v1/objects
Authorization: Bearer <session token>
Content-Type: application/json

{
  "object": {
    "kind": "block:wood",
    "transform": {"x": 1, "y": 0, "z": 0, "rotation": 0},
    "properties": {}
  },
  "expected_revision": 12
}
```

- 省略 `object.id` 时由 Home 分配新 ID；带 ID 时更新或创建该 ID。
- `expected_revision` 不匹配返回 `409 REVISION_CONFLICT`，响应带当前 `revision`；客户端应重新读取快照。
- `DELETE /v1/objects/{id}` 同样可以带 `expected_revision`。
- 每次成功变更递增修订并唤醒 `/v1/events` 的长轮询。
- `presence` 每 10 秒左右发送一次；Home 会过期过时的在线状态。

## 7. 邀请、成员和审计

创建邀请：

```http
POST /v1/admin/invites
Authorization: Bearer <owner-control token>

{"role": "builder", "expires_in_ms": 3600000, "max_uses": 3}
```

响应中的 `code` 只出现一次，格式为 `inv_<24 hex>.<32–64 hex>`。数据库只保存 secret 的 SHA-256。`role` 只能是 `builder` 或 `visitor`，不能创建 owner。

launcher 在第一次创建邀请时会把 loopback 宿主重启到 `0.0.0.0:<port>`，并在自己的响应里附加 `lan_started`、`base_url` 和加入链接。链接格式为：

```text
interisle://home/join?url=<percent-encoded base_url>&invite=<percent-encoded code>
```

加入端也接受 `http://host:port invite_code` 的粘贴形式。链接和邀请码都只是传输格式；真正的加入凭证仍是访客账号密钥签名的 `HomeJoinProof`。

成员操作请求体分别是：

```json
{"role": "visitor"}
{"reason": "optional"}
```

owner 不能降级或踢出自己。被踢出的成员会话立即失效；被封禁的账号不能用新邀请码或 `rejoin` 加入。

## 8. 包清单

`world.json` 的关键字段：

```json
{
  "format": "interisle-world/1",
  "id": "home.default",
  "version": "0.1.0",
  "name": {"en": "Home", "zh-cn": "家园"},
  "description": {"en": "A calm, editable island home."},
  "content_hash": "sha256:...",
  "renderers": [
    {"kind": "godot", "entry": "render/godot/project.godot"},
    {"kind": "web", "entry": "render/web/index.html"}
  ],
  "files": {
    "render/web/index.html": "<sha256 hex>",
    "render/web/home.js": "<sha256 hex>"
  }
}
```

约束：

- `content_hash` 是移除自身后的规范 JSON 的 SHA-256，前缀 `sha256:`。
- `renderers` 1–4 项，kind 不重复，entry 必须在 `render/` 下且被 `files` 覆盖。
- 包内最多 512 个文件；拒绝绝对路径、`..`、符号链接和重复条目。
- 世界 ID 最长 64 字节，只允许小写字母、数字、`-`、`_`、`.`。
- 渲染器资源通过 `/world/<entry>` 提供；HTML 只允许同源脚本、样式和 data/blob 图片，并限制可嵌入的 launcher 来源。

构建和校验：

```bash
cargo run -p xtask -- world pack worlds/home-default worlds/home-default.islworld
cargo run -p xtask -- world verify worlds/home-default.islworld
cargo run -p xtask -- world inspect worlds/home-default.islworld
```

## 9. 运行 `isl-home`

```bash
isl-home \
  --listen 127.0.0.1:7431 \
  --data profiles/me/home/world.sqlite \
  --world-dir profiles/me/home/world \
  --package profiles/me/home/world.islworld \
  --key profiles/me/home/home.key \
  --owner-token-file profiles/me/home/owner.token \
  --world-id home.default \
  --name "Home" \
  --content-hash sha256:... \
  --owner-did did:key:z...
```

`--owner-did` 和 `--content-hash` 必填。owner token 文件必须是 owner-only；不要通过命令行、环境变量或日志传递 token 值。launcher 负责生成这些路径并在后台监督进程。

## 10. 失败关闭

| 情况 | 行为 |
|---|---|
| 无 bearer token | 401 `UNAUTHORIZED` |
| 角色不足 | 403 `FORBIDDEN` |
| 未知路由/方法 | 404 或 409 `METHOD_NOT_ALLOWED` |
| 修订冲突 | 409 `REVISION_CONFLICT`，带当前 revision |
| 过期/重用挑战 | 400，拒绝加入 |
| 内容哈希不匹配 | 400，拒绝加入 |
| 被踢/被封禁 | 401/403，不能重连 |
| 请求体包含未知字段 | 400 `INVALID_REQUEST` |
| 渲染器 token 过期或撤销 | 401，渲染器停止拉取 |

Home 不与 coordinator、isle、visitor 或仓库共享签名域；任何 Home 签名对象都不能用于凭证签发、兑换或提取。

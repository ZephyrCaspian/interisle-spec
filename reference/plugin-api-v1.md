# 插件 API v1

- 依据：ADR 0021、ADR 0025
- 适用：页面、代码、主题、语言、家园、渲染器六类本地插件包

本规范只描述 launcher 插件包、声明式 UI、宿主协议和限制，不改变协议线格式、凭证或签名域。

## 1. 包格式

插件包是扩展名为 `.islpkg` 的 ZIP：

```text
manifest.json
ui/page.json              # page/code
ui/theme.json             # theme
ui/locale.json            # locale
logic/plugin.wasm         # code
bin/isl-home.exe          # home，入口由 entry.home 声明
bin/home-renderer.exe     # renderer，入口由 entry.renderer 声明
world/home.islworld       # home，入口由 entry.world 声明
assets/...                # 可选，仅 PNG/WebP
signatures/manifest.sig   # 可选，Ed25519
```

安装器拒绝：绝对路径、`..`、符号链接、重复条目、加密条目、过多文件、单文件/总解压体积超限、未知顶层条目（除 `manifest.json`、`ui/`、`logic/`、`assets/`、`bin/`、`world/`、`signatures/`）。

限制（v1）：最多 1024 个条目；包文件与总解压各 512 MiB；单文件 256 MiB；manifest 1 MiB；路径长度 240 字节；WASM 8 MiB；图片单张 2 MiB；Home 原生宿主和嵌套 `.islworld` 各 16 MiB；渲染器原生可执行文件 256 MiB（导出的 Godot 运行时约 100 MiB）。

## 2. manifest.json

```json
{
  "format": "interisle-plugin/1",
  "id": "community.example.notes",
  "version": "1.0.0",
  "kind": "code",
  "name": { "zh": "便签", "en": "Notes" },
  "description": { "zh": "本地便签示例", "en": "Local notes example" },
  "api_version": 1,
  "entry": { "ui": "ui/page.json", "wasm": "logic/plugin.wasm" },
  "permissions": ["storage.self"],
  "publisher": { "did": "did:key:z6Mk...", "name": "Example" },
  "integrity": {
    "algorithm": "sha256",
    "files": {
      "ui/page.json": "<hex>",
      "logic/plugin.wasm": "<hex>"
    }
  }
}
```

字段规则：

| 字段 | 规则 |
|---|---|
| `format` | 必须是 `interisle-plugin/1` |
| `id` | 小写字母/数字/`.`/`-`，2–96 字符；不得以 `core.`、`official.`、`interisle.` 开头 |
| `version` | SemVer 三段 |
| `kind` | `page` / `code` / `theme` / `locale` / `home` / `renderer` |
| `name`/`description` | 语言键 `zh`/`en` 必填；其他语言可选；纯文本，不含 HTML |
| `api_version` | 主版本必须为 `1`；其他拒绝安装 |
| `entry.ui` | `page`/`code` 必填；其余不填 |
| `entry.wasm` | `code` 必填；其余不填 |
| `entry.home` | `home` 必填；必须是 `bin/` 下的文件 |
| `entry.world` | `home` 必填；必须是 `world/` 下的 `.islworld` 文件 |
| `entry.renderer` | `renderer` 必填；必须是 `bin/` 下的文件 |
| `renderer.kind` | `renderer` 必填；小写标识符，必须与世界包的 `renderers[].kind` 匹配（当前为 `godot`） |
| `permissions` | 只允许 `storage.self` 和 `native.execute`；后者只对 `home`、`renderer` 有效 |
| `publisher` | 可选；`did` 必须是 `did:key:` 字符串 |
| `integrity.files` | 覆盖所有会被加载的文件；SHA-256 小写十六进制 |

`id` 唯一标识插件；同一 ID 的更高 SemVer 作为更新，必须保留上一版本用于回滚。内置页面 ID 为 `builtin.launch`、`builtin.downloads`、`builtin.extensions`、`builtin.settings`、`builtin.more`，第三方不得使用 `builtin.` 前缀。

### 2.1 家园插件（home）

`kind: "home"` 把本地非经济 Home 作为一个插件分发。示例 manifest：

```json
{
  "format": "interisle-plugin/1",
  "id": "home.default",
  "version": "1.0.0",
  "kind": "home",
  "name": { "zh": "家园", "en": "Home" },
  "description": { "zh": "本地运行的非经济家园世界", "en": "A locally hosted, non-economic Home world" },
  "api_version": 1,
  "entry": {
    "home": "bin/isl-home.exe",
    "world": "world/home-default.islworld"
  },
  "permissions": ["native.execute", "storage.self"],
  "integrity": {
    "algorithm": "sha256",
    "files": {
      "bin/isl-home.exe": "<hex>",
      "world/home-default.islworld": "<hex>"
    }
  }
}
```

- 包内 `bin/` 和 `world/` 只能包含 manifest 声明的两个文件；`entry.world` 必须通过 `isl_world_package::verify_package_bytes`。
- launcher 在安装时和每次启动前都校验这两个文件的 SHA-256，不信任磁盘上的已安装副本。
- 导入的 Home 包必须带受信发布者签名；随包市场的 Home 由 `market/catalog.json` 的 SHA-256 固定，属于分发信任。
- Home 插件更新若改变 `entry.world` 的内容哈希，launcher 会把同一 `world_id` 的旧数据库、`home.key`、`owner.token` 和备份迁移到新的内容寻址目录，并用 `--migrate-from-content-hash` 授权 `isl-home` 采纳旧状态。world ID、Home DID、owner DID 或旧哈希不匹配时，`isl-home` 拒绝打开。
- Home 插件不会创建经济会话，也不提供 `/v1/issue`、`/v1/extract`、`/v1/claim` 或 `/v1/warehouse`。
- `xtask plugin pack-home` 生成该包，`xtask plugin catalog` 生成市场目录。

### 2.2 渲染器插件（renderer）

`kind: "renderer"` 把一个原生世界渲染器作为独立插件分发。它与 `home` 服务插件分离：Home 负责 `interisle.world/v1` 世界状态，渲染器只负责显示和本地交互。示例 manifest：

```json
{
  "format": "interisle-plugin/1",
  "id": "renderer.godot",
  "version": "1.0.0",
  "kind": "renderer",
  "name": { "zh": "Godot 渲染器", "en": "Godot Renderer" },
  "description": { "zh": "随包自带 Godot 运行时", "en": "Bundled Godot runtime" },
  "api_version": 1,
  "entry": { "renderer": "bin/isl-home-renderer.exe" },
  "renderer": { "kind": "godot" },
  "permissions": ["native.execute"],
  "integrity": {
    "algorithm": "sha256",
    "files": {
      "bin/isl-home-renderer.exe": "<hex>"
    }
  }
}
```

- 包内 `bin/` 只能包含 manifest 声明的渲染器可执行文件；`entry.renderer` 必须指向 `bin/` 下的文件，且 `renderer.kind` 必须出现在所运行世界包的 `renderers[]` 中。
- launcher 在安装时和每次启动前都校验可执行文件的 SHA-256；导入的渲染器包必须带受信发布者签名，随包市场中的渲染器由 `market/catalog.json` 的 SHA-256 固定。
- launcher 以独立子进程启动导出的可执行文件，并在 `--` 之后传入 `--home-url`、`--token-file`、`--manifest`、`--display-name`、`--bridge-url`、`--bridge-token-file`。`--token-file` 是 owner-only 的短期渲染器会话 token，不是 owner-control token。
- 游戏内管理员菜单通过 `--bridge-url` 指向的 loopback HTTP 通道访问启动器。通道只接受 `POST /v1/home/admin`、每次启动随机生成的 Bearer 密钥、固定操作白名单和有限大小的 JSON；启动器校验后用自己的 owner-control token 调用 Home。owner-control token 不进入渲染器进程、环境变量或日志。
- 找不到渲染器插件时，launcher 只在开发环境的源码世界包中回退到本机 Godot 运行 `render/godot/project.godot`；便携包携带 `renderer.godot`，玩家不需要安装 Godot。
- `xtask plugin pack-renderer` 生成该包，`xtask plugin catalog` 生成市场目录。

## 3. 声明式 UI（page.json）

```json
{
  "schema": 1,
  "title": { "zh": "便签", "en": "Notes" },
  "body": [
    {
      "type": "section",
      "title": { "zh": "本地便签", "en": "Local notes" },
      "children": [
        { "type": "input", "id": "text", "label": { "zh": "内容", "en": "Text" } },
        {
          "type": "button",
          "label": { "zh": "保存", "en": "Save" },
          "action": { "type": "invoke", "command": "save" }
        },
        { "type": "text", "bind": "state.status" }
      ]
    }
  ]
}
```

### 3.1 组件白名单

| type | 必需字段 | 可选字段 | 说明 |
|---|---|---|---|
| `section` | `children` | `title`, `id` | 分组容器 |
| `card` | `children` | `title`, `id` | 卡片容器 |
| `text` | `value` 或 `bind` | `tone` | 纯文本，不解析 HTML |
| `button` | `label`, `action` | `id`, `tone`, `disabled` | 点击派发动作 |
| `input` | `id` | `label`, `placeholder`, `value`, `multiline` | 写回本地 UI state |
| `select` | `id`, `options` | `label`, `value` | `options` 为 `{value,label}` 数组 |
| `switch` | `id` | `label`, `value` | 布尔值 |
| `list` | `bind`, `item` | `empty` | `bind` 指向数组；`item` 是模板节点 |
| `table` | `bind`, `columns` | `empty` | `columns` 为 `{key,label}` |
| `tabs` | `tabs` | `id` | `tabs` 为 `{id,label,children}` |
| `badge` | `value` 或 `bind` | `tone` | 小标签 |
| `progress` | `bind` | `label` | 0–100 |
| `empty` | `title` | `description` | 空状态 |
| `divider` | 无 | 无 | 分隔线 |
| `image` | `src` | `alt`, `width`, `height` | 只允许包内 PNG/WebP |

- 容器深度最多 8 层，节点总数最多 500。
- `bind` 只允许 `state.<path>`；路径由字母/数字/`_`/`.` 组成，不接受表达式、函数或数组下标之外的内容。
- 本地化文本是 `{ "zh": "...", "en": "..." }` 对象；缺失语言回退 `zh`，再回退 `en`。
- 禁止字段：`html`、`css`、`style`、`script`、`onclick`、`href`、`srcset`、`class`、`v-html`。未知字段和未知组件直接拒绝该页面，不静默忽略。

### 3.2 动作

```json
{ "type": "set", "path": "state.text", "value": "..." }
{ "type": "toast", "message": { "zh": "已保存", "en": "Saved" } }
{ "type": "navigate", "tab": "details" }
{ "type": "invoke", "command": "save" }
```

- `invoke` 只允许 `code` 插件；`page` 插件使用 `set`/`toast`/`navigate`。
- `set` 只能写插件本地 UI state，不能写 launcher/core state。
- `navigate` 只能切到同一页面的 tab。
- 动作不接收 URL、文件路径、shell 命令或任意 Tauri 命令名。
- 渲染器不把 `invoke` 暴露给 DOM；所有动作由 launcher core 校验。

## 4. 主题（theme.json）

```json
{
  "schema": 1,
  "id": "community.theme.dusk",
  "name": { "zh": "暮色", "en": "Dusk" },
  "tokens": {
    "--bg": "#111318",
    "--panel": "#181b22",
    "--text": "#e8eaf0",
    "--accent": "#7aa2f7"
  }
}
```

允许 token 为 `--bg`、`--bg-soft`、`--panel`、`--panel-strong`、`--text`、`--text-muted`、`--border`、`--accent`、`--accent-strong`、`--danger`、`--ok`、`--radius`、`--font-ui`。值只接受十六进制颜色、`rgb()/rgba()`、有限长度（`px`/`rem`/`em`）、字体族列表；拒绝 `url()`、`var()`、`expression()`、`calc()` 和任何包含 `;`/`}` 的值。非法值回退默认主题。

## 5. 语言（locale.json）

```json
{
  "schema": 1,
  "id": "community.locale.ja",
  "name": { "zh": "日语", "en": "Japanese" },
  "locale": "ja",
  "strings": { "launch.play": "プレイ", "common.cancel": "キャンセル" }
}
```

- 只允许字符串到字符串；嵌套最多 3 层或使用扁平键。
- 不允许 HTML、URL、脚本；值为纯文本，最长 500 字符。
- 不允许覆盖 `security.*`、`consent.*`、`plugin.install.*` 等安全/同意文案；这些键由官方内置文案固定。
- 缺失键回退官方 `zh`，再回退 `en`。

## 6. 代码插件（WASM）

`logic/plugin.wasm` 导出：

```wat
(memory (export "memory") 1)
(func (export "interisle_alloc") (param i32) (result i32))
(func (export "interisle_invoke") (param i32 i32) (result i32))
(func (export "interisle_result_len") (result i32))
```

调用顺序：

1. 宿主把 UTF-8 JSON `{"command":"save","input":{...},"state":{...}}` 写入 `interisle_alloc(len)` 返回的包内指针。
2. 宿主调用 `interisle_invoke(ptr, len)`；返回值是结果 JSON 的包内指针。
3. 宿主调用 `interisle_result_len()` 取得结果长度，再读取并解析 JSON。
4. 返回是 UTF-8 JSON：`{"state":{...},"toast":"..."}`；只允许这两个字段，`state` 必须是对象，`toast` 必须是字符串。
5. 失败返回 `{"error":"..."}`；宿主把它转换为 `plugin_error`，插件进程继续运行。

`interisle_alloc`、`interisle_invoke`、`interisle_result_len` 和 `memory` 都是必需导出；缺失任一导出拒绝加载。宿主函数从 `env` 命名空间导入，例如：

```wat
(import "env" "isl_kv_get" (func $kv_get (param i32 i32 i32 i32) (result i32)))
```

- 不允许 WASI、文件系统、网络、环境变量、线程、共享内存或宿主 Tauri API。
- 宿主不把包目录以外路径提供给插件；KV 数据限制在插件自己的 `data/<plugin-id>/`。

### 6.1 宿主函数

| 函数 | 签名 | 说明 |
|---|---|---|
| `isl_log` | `(level, ptr, len)` | 级别 0–3；日志不包含敏感数据 |
| `isl_kv_get` | `(key_ptr,key_len,out_ptr,out_cap) -> i32` | 返回长度；没有值为 0；超容量返回负值 |
| `isl_kv_set` | `(key_ptr,key_len,val_ptr,val_len) -> i32` | 0 成功；非 0 失败 |
| `isl_now` | `() -> i64` | Unix 毫秒 |
| `isl_random` | `(ptr,len)` | 填充随机字节 |

KV 键最长 128 字节，值最大 64 KiB，插件总配额 1 MiB。键只允许 `[A-Za-z0-9._-]`。

### 6.2 限制

- 单次调用 2 秒墙钟、fuel 10^9、线性内存 64 MiB、输出 256 KiB。
- 超时/超限/陷阱：杀宿主进程，插件标记失败；launcher 继续运行，可重试或禁用。
- 宿主按需启动，空闲 60 秒可退出；崩溃后下次调用重启。

## 7. 宿主协议

launcher 与 `isl-plugin-host` 通过 stdio 逐行 JSON 通信：

```json
{"id":1,"method":"hello","params":{"api_version":1,"plugin_id":"community.example.notes","package_dir":"...","data_dir":"...","permissions":["storage.self"]}}
{"id":1,"result":{"ok":true}}
{"id":2,"method":"invoke","params":{"command":"save","input":{"text":"..."},"state":{},"timeout_ms":2000}}
{"id":2,"result":{"state":{"status":"saved"},"toast":"已保存"}}
{"id":3,"method":"shutdown"}
```

- `hello` 必须首先成功；`api_version` 不为 1 时宿主拒绝并退出。
- `package_dir`/`data_dir` 由 launcher 校验后再传入；宿主不接收 profile、密钥、token 或 visitor 地址。
- `timeout_ms` 省略时为 2000，允许 100–10000。
- 错误响应为 `{"id":N,"error":{"code":"...","message":"...","fatal":true|false}}`。`plugin_error` 表示插件自己返回了错误，宿主继续运行；`plugin_timeout`、`limit_exceeded`、`plugin_trap` 等致命错误会让宿主在响应后退出，launcher 下次调用时重启。
- 未知 method、非 JSON 行：宿主返回错误或退出；launcher 重启。
- 宿主 stdout 只输出协议 JSON；日志写 stderr。

## 8. 签名与完整性

- `integrity.files` 必须覆盖包内除 `manifest.json` 和 `signatures/manifest.sig` 以外的所有文件（包括 `page.json` 引用的图片）；安装时逐文件校验，缺失或不匹配拒绝安装。
- `signatures/manifest.sig` 可选，内容是纯 multibase（`z` 开头的 base58btc）Ed25519 签名，签名域为 `interisle-plugin-manifest/1`，签名对象是规范化 manifest（去除顶层 `signatures`、无多余空白、按键排序）。
- 签名有效但发布者不在信任列表：显示“签名有效，发布者未受信”，仍需显式确认。
- 无签名：显示“未签名”，必须显式确认；不提供静默安装。
- 确认安装时，launcher 以整个 `.islpkg` 的 SHA-256 绑定用户确认的包；文件在确认后变化则重新检查，不安装。
- v1 不校验证书链、时间戳或吊销列表。
- 安装时非法主题 token 或语言键直接拒绝；运行期若已启用主题无法读取或校验失败，前端回退内置默认主题，不执行未校验值。

## 9. 安装与注册表

```text
<data-root>/plugins/
  registry.json
  packages/<plugin-id>/<version>/
  data/<plugin-id>/
```

`registry.json` 只由 launcher 写入，记录已安装版本、启用状态、安装时间、签名状态和当前选择版本。内置项不写入注册表，不能被卸载或覆盖。

安装步骤：

1. 读取 ZIP 中央目录，拒绝危险条目和超限。
2. 解压到临时目录。
3. 解析 manifest，校验 ID/版本/kind/api_version/权限/入口。
4. 校验完整性；存在签名时验证。
5. 原子 rename 到 `packages/<id>/<version>`。
6. 写注册表；失败则回滚并清理临时目录。

卸载默认保留 `data/<id>/`；删除数据是单独确认。回滚切换注册表中的当前版本，不删除新版本文件。

## 10. 明确不做（v1）

- 在线市场、远程更新、自动更新、付费、评分（随包离线市场已实现）。
- 第三方 native 代码（除受信签名的 `home` / `renderer` 插件）、WASI、网络、文件系统、进程 API。
- 任意 HTML/CSS/JS/SVG 页面或外部 URL 图片。
- 插件替换解锁/恢复/安装确认/安全提示。
- 多设备插件同步。

# Interisle

Interisle 是一套让物品跨独立游戏服务器携带的协议。它不是某个游戏的插件，而是一份规范和一组可替换的实现。

> **状态：** 协议 v1 规范已发布（`spec-v1.0.0`，2026-11-02）。参考实现正在开发中，尚未公开。

## 这是什么

Interisle 把“世界内的物品”和“跨服携带的所有权”分开：

- 游戏服务器（isle）拥有世界、会话和背包；
- 玩家仓库只保存签名的物品凭证；
- coordinator 在每个 claim domain 内防止凭证双花和实例分叉；
- 目标服务器通过适配器决定如何物化物品。

完整背景见[问题与约束](whitepaper/01-problem.md)，系统结构见[架构](whitepaper/02-architecture.md)。

## 规范内容

- [白皮书：问题与约束](whitepaper/01-problem.md)
- [白皮书：架构](whitepaper/02-architecture.md)
- [白皮书：安全模型](whitepaper/03-security.md)
- [白皮书：治理](whitepaper/04-governance.md)
- [Home 协议 v1（`interisle.world/v1`）](reference/home-protocol-v1.md)
- [插件 API v1](reference/plugin-api-v1.md)
- [术语表](reference/glossary.md)
- [路线图](ROADMAP.md)

## 版本与状态

- 当前规范版本：**v1.0.0**（tag `spec-v1.0.0`）。
- 协议 v1 是唯一运行路径；早期草案中的 relay、mini-isle、catalog/mapping、能力凭证和 split/merge 不属于 v1。
- 参考实现尚未公开；本仓库只发布协议规范，不包含实现代码。
- 破坏性变更走新 major 版本；规范版本与实现版本独立编号。
- 正文中出现的 ADR 编号是内部设计记录引用；ADR 文档暂未随本规范发布。

## 许可

- 规范正文：CC BY 4.0，见 [LICENSE](LICENSE)。
- 规范中的代码示例：Apache-2.0，见 [LICENSE-CODE](LICENSE-CODE)。
- “Interisle”名称、标识和品牌资产不在上述许可范围内，见 [TRADEMARK.md](TRADEMARK.md)。

## 引用

见 [CITATION.cff](CITATION.cff)。

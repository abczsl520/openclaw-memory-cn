<div align="center">

# 🧠 openclaw-memory-cn

**让 OpenClaw 的中文记忆搜索从「废物」变「能用」**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![OpenClaw](https://img.shields.io/badge/OpenClaw-2026.3+-blue.svg)](https://github.com/openclaw/openclaw)
[![Ollama](https://img.shields.io/badge/Ollama-compatible-green.svg)](https://ollama.ai)

*55% → 100% 命中率 | 0 代码修改 | 5 分钟部署*

[English](README_EN.md) · [技术分析](#-核心发现fts5-的中文-bug) · [快速开始](#-快速开始) · [效果对比](#-效果对比)

</div>

---

## 😡 你是不是也遇到过这种情况？

```
你: 我项目组立项要去哪个会议室来着？
AI: 抱歉，我在记忆中没有找到相关信息。

你: ？？？我明明上周跟你说过啊！！
```

**这不是你的问题，是 SQLite FTS5 的 bug。**

OpenClaw 用 FTS5 做全文搜索，但它的 `unicode61` 分词器会把 `项目组刚立项选定广东省研发中心` 当成**一整个 token**。你搜 `立项`？不好意思，匹配不上。

本项目用**纯配置方案**（0 代码修改）解决这个问题。

## 📊 效果对比

| 指标 | 优化前 | 优化后 | 提升 |
|------|--------|--------|------|
| 中文搜索命中率 | 55% | **100%** | +82% |
| 零召回查询 | 10/20 | **0/20** | 🎉 |
| 最高检索分数 | 0.67 | **0.80** | +19% |
| 记忆体积 | ~780KB | ~460KB | -41% |

> 测试环境：OpenClaw 2026.3.2 + qwen3-embedding:0.6b + 80个记忆文件

## 🚀 快速开始

### 30 秒版本（老手）

```bash
# 应用搜索参数 + memoryFlush prompt
openclaw config patch < config/memory-search.json
openclaw config patch < config/memory-flush-prompt.json

# 给项目文件加标签 + 重建索引
python3 scripts/add-tags.py ~/path/to/workspace/memory/projects/
openclaw memory index --force
```

### 5 分钟版本（新手）

**Step 1: 诊断**

```bash
git clone https://github.com/abczsl520/openclaw-memory-cn.git
cd openclaw-memory-cn
./scripts/diagnose.sh
```

脚本会告诉你：
- ❌ FTS5 是否有中文分词 bug
- ❌ 哪些文件过大需要压缩
- ❌ 哪些项目文件缺少标签

**Step 2: 优化搜索配置**

```bash
openclaw config patch < config/memory-search.json
```

这会设置：向量搜索权重 0.75（因为 FTS5 中文不可靠）、更小的 chunk size（帮助弱模型精确匹配）、MMR 去重、时间衰减等。

**Step 3: 优化写入格式**

```bash
openclaw config patch < config/memory-flush-prompt.json
```

以后 memoryFlush 写日志时会自动：
- 第一行加 `<!-- tags: 关键词 -->` 标签
- 中文关键词空格分隔（`项目组 立项` 而非 `项目组立项`）
- 控制在 5KB 以内

**Step 4: 清理和标记**

```bash
# 给现有项目文件批量加标签
python3 scripts/add-tags.py ~/workspace/memory/projects/

# 压缩膨胀的日志（>8KB 的压缩到 <5KB）
python3 scripts/compress-logs.py ~/workspace/memory/ --max-kb 5

# 清理垃圾文件
./scripts/cleanup.sh ~/workspace/memory/

# 重建索引
openclaw memory index --force
```

**Step 5: 自动维护（可选但推荐）**

设置每周 cron 自动压缩日志 + 清理 archive + 补标签 + 重建索引：

复制 `templates/cron-job.json` 的内容，通过 OpenClaw 的 cron 功能添加。

## 🔬 核心发现：FTS5 的中文 Bug

### 问题

SQLite FTS5 默认的 `unicode61` 分词器把**连续中文字符视为一个 token**：

```sql
-- "项目组刚立项" 在索引中是 1 个 token
-- 搜索 "立项" → 0 results ❌
SELECT * FROM chunks_fts WHERE chunks_fts MATCH '立项';  -- → 0 ❌
```

### 证据

从 FTS5 词汇表中查到的实际 token：

| 原文 | FTS5 token | 字符数 |
|------|-----------|--------|
| 用户明天要去会议室 | `用户明天要去会议室` | 7 |
| 确定华南区研发中心三楼会议室 | `确定华南区研发中心三楼会议室` | 13 |
| 也叫小王 | `也叫小王` | 5 |

### Workaround

在关键词之间加空格，让 FTS5 把它们索引为独立 token：

```
❌ "项目组刚立项，选定广东省研发中心"  → 1个token
✅ "项目组 刚立项，选定 广东省 研发中心" → 4个token，搜索可命中
```

详细技术分析见 [docs/fts5-unicode61-bug.md](docs/fts5-unicode61-bug.md)

## 📐 三层记忆架构

```
workspace/
├── MEMORY.md (P0)           ← 核心事实，每次启动必读（<2KB）
│   ├── 用户关键信息
│   ├── 服务器/凭证（指针）
│   └── 核心工作规则
└── memory/
    ├── projects/ (P1)       ← 项目档案，带 tags 标签
    ├── lessons/ (P1)        ← 经验教训，搜索命中率最高的文件类型
    ├── YYYY-MM-DD.md (P2)   ← 每日日志（自动生成+自动分词）
    └── archive/             ← 完整备份（30天自动清理）
```

模板文件见 [templates/](templates/)

## ⚙️ 推荐参数

| 参数 | 值 | 为什么 |
|------|-----|--------|
| `vectorWeight` | 0.75 | FTS5 中文不可靠，向量主导 |
| `textWeight` | 0.25 | FTS5 只对英文和分词后的中文有效 |
| `chunking.tokens` | 250 | 小 chunk 帮助弱模型精确匹配 |
| `chunking.overlap` | 60 | 保证上下文连贯 |
| `minScore` | 0.15 | 小模型分数偏低，阈值要低 |
| `halfLifeDays` | 90 | 项目配置长期有效 |
| `MMR lambda` | 0.7 | 适度去重 |

## 🤔 适用范围

| ✅ 适用 | ❌ 不适用 |
|---------|----------|
| OpenClaw 2026.3+ | 其他 AI 框架 |
| Ollama 本地小模型 | OpenAI/Voyage 云端大模型* |
| 中文为主的使用场景 | 纯英文场景（FTS5英文正常） |
| qwen3-embedding:0.6b | text-embedding-3-large |
| nomic-embed-text | voyage-3 |
| mxbai-embed-large | |

*云端大模型用户：文件结构和 memoryFlush prompt 仍然适用，但搜索参数需要调整（大模型不需要这么极端的 vectorWeight）。

## 📈 为什么这些优化有效

```
写入时优化（memoryFlush prompt）
    ↓
  tags 标签 → FTS5 精确命中 ✅
  中文空格 → FTS5 正确分词 ✅
  大小控制 → chunk 更精确 ✅
    ↓
搜索时优化（config 参数）
    ↓
  vectorWeight 0.75 → 绕过 FTS5 中文缺陷 ✅
  小 chunk → 弱模型更容易匹配 ✅
  低 minScore → 不漏掉低分但相关的结果 ✅
  MMR → 不重复返回相似内容 ✅
    ↓
维护时优化（cron job）
    ↓
  压缩旧日志 → 减少噪音 ✅
  补 tags → 新文件也有标签 ✅
  重建索引 → 保持索引新鲜 ✅
```

## 🤝 贡献

欢迎 Issue 和 PR！特别欢迎：

- 📊 其他小模型（nomic-embed-text、mxbai 等）的测试数据
- 🇯🇵🇰🇷 日文/韩文的 FTS5 分词问题验证（理论上同样受影响）
- 🔧 更好的中文分词 workaround
- 📝 其他语言的翻译

## ⭐ Star History

如果这个项目帮到了你，请给个 Star ⭐ 让更多人看到！

## 📄 License

[MIT](LICENSE)

---

<div align="center">

*从实际生产环境中总结，所有数据均为真实测试结果。*

**[OpenClaw](https://github.com/openclaw/openclaw)** · **[Ollama](https://ollama.ai)** · **[SQLite FTS5](https://www.sqlite.org/fts5.html)**

</div>

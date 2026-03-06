# FTS5 unicode61 中文分词 Bug 技术分析

## 问题描述

OpenClaw 使用 SQLite FTS5 全文搜索作为混合搜索的文本分支。FTS5 默认使用 `unicode61` 分词器。

`unicode61` 分词器的规则是：
- 根据 Unicode 6.1 字符类别分词
- **字母和数字字符连续出现时，视为同一个 token**
- CJK 统一汉字（U+4E00-U+9FFF）被归类为"字母"

这意味着：**连续的中文字符会被当作一个巨大的 token**。

## 复现

```sql
-- 查看 FTS5 实际索引的 token
SELECT term, doc FROM chunks_fts_vocab WHERE col = '*' AND term LIKE '%立项%';
-- 结果: "用户项目组刚立项" — 整段话是1个token！

-- 精确匹配：失败
SELECT COUNT(*) FROM chunks_fts WHERE chunks_fts MATCH '立项';
-- → 0 ❌

-- 前缀匹配：可以（因为"立项"恰好在某些token末尾）
SELECT COUNT(*) FROM chunks_fts WHERE chunks_fts MATCH '立项*';
-- → 2 ✅ (但这是运气)

-- LIKE 搜索：可以（因为是子串匹配，不走索引）
SELECT COUNT(*) FROM chunks_fts('text') WHERE text LIKE '%立项%';
-- → 4 ✅ (但是全表扫描，性能差)
```

## 影响范围

所有连续书写的中文文本都受影响：

```
"确定华南区研发中心三楼会议室" → 1个token (15个字符)
"也叫小王"                 → 1个token (5个字符)
"抖音星图达人"               → 1个token (6个字符)
```

但被标点符号或空格分隔的中文不受影响：

```
"用户，你好" → "用户" + "你好" (2个token，逗号分隔)
"Telegram: @dev" → 正常分词
```

## OpenClaw 的修复状态

- **Issue #17672**（已合并）：修复了 `buildFtsQuery` 函数的正则表达式，从 `\w+` 改为 `[\p{L}\p{N}_]+`，让查询端能正确提取中文 token
- **但**：索引端仍然使用 `unicode61` 分词器，索引中的 token 仍然是粘连的
- FTS5 的 `tokenize` 参数在 OpenClaw 中不可配置（写死在代码里）

## Workaround

### 方法1：在写入时加空格（推荐）

在中文关键词之间加空格，让 `unicode61` 把它们索引为独立 token：

```markdown
❌ 项目组刚立项，选定广东省研发中心
✅ 项目组 刚立项，选定 广东省 研发中心
```

可以通过自定义 `memoryFlush.prompt` 让 OpenClaw 自动生成分词格式的日志。

### 方法2：降低 FTS5 权重

既然 FTS5 对中文不可靠，就让向量搜索主导：

```json
{
  "vectorWeight": 0.75,
  "textWeight": 0.25
}
```

### 方法3：添加 tags 标签行

在每个文件开头加上带空格的标签行：

```markdown
<!-- tags: 立项, 会议室, 周会, 研发中心 -->
# 立项攻略
...
```

标签中的关键词会被逗号和空格分隔，FTS5 可以正确索引。

## 理想方案

OpenClaw 应该支持配置 FTS5 的 `tokenize` 参数，例如：

```sql
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text,
  tokenize = "unicode61 categories 'L* N*' separators ''"
);
```

或者支持使用 ICU 分词器（需要 SQLite ICU 扩展）：

```sql
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text,
  tokenize = "icu zh_CN"
);
```

建议向 OpenClaw 项目提 Issue 请求此功能。

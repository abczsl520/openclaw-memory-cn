# 诊断指南

## 如何判断你的记忆系统需要优化

### Step 1：检查 FTS5 分词

```bash
# 查看你的 FTS5 是否存在中文分词问题
sqlite3 ~/.openclaw/memory/main.sqlite \
  "SELECT term FROM chunks_fts_vocab WHERE col='*' AND length(term) > 8 LIMIT 20;"
```

如果看到超长的中文字符串（如 `项目刚立项选定广东省`），说明 unicode61 bug 影响了你。

### Step 2：测试搜索质量

在 OpenClaw 中运行：
```
/memory search 你最常查的中文关键词
```

如果返回 0 条或不相关结果，说明需要优化。

### Step 3：检查文件膨胀

```bash
# 看看记忆文件大小分布
find ~/.openclaw/workspace/memory -name '*.md' -exec ls -lS {} + | head -20
```

超过 30KB 的日志文件需要压缩。

### Step 4：运行诊断脚本

```bash
./scripts/diagnose.sh
```

脚本会自动检测所有问题并给出建议。

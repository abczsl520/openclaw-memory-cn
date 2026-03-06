<div align="center">

# 🧠 openclaw-memory-cn

**Make OpenClaw's Chinese memory search actually work**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![OpenClaw](https://img.shields.io/badge/OpenClaw-2026.3+-blue.svg)](https://github.com/openclaw/openclaw)
[![Ollama](https://img.shields.io/badge/Ollama-compatible-green.svg)](https://ollama.ai)

*55% → 100% hit rate | Zero code changes | 5-minute setup*

[中文文档](README.md)

</div>

---

## The Problem

If you use OpenClaw + Ollama with a small embedding model for Chinese content, you've probably experienced this:

```
You: Which hospital did my wife choose for prenatal care?
AI: Sorry, I couldn't find any relevant information in memory.

You: I LITERALLY TOLD YOU LAST WEEK!
```

**It's not your fault. It's an SQLite FTS5 tokenization bug.**

FTS5's `unicode61` tokenizer treats consecutive CJK characters as a **single token**. So `项目组刚立项选定广东省研发中心` (13 characters about pregnancy + hospital) becomes ONE token. Searching for `立项` (pregnancy)? No match.

This project fixes it with **pure configuration** — zero code changes.

## Results

| Metric | Before | After |
|--------|--------|-------|
| Chinese search hit rate | 55% | **100%** |
| Zero-recall queries | 10/20 | **0/20** |
| Max retrieval score | 0.67 | **0.80** |
| Memory size | ~780KB | ~460KB (-41%) |

## Quick Start

```bash
# Clone
git clone https://github.com/abczsl520/openclaw-memory-cn.git
cd openclaw-memory-cn

# Diagnose your memory system
./scripts/diagnose.sh

# Apply optimized search config
openclaw config patch < config/memory-search.json
openclaw config patch < config/memory-flush-prompt.json

# Tag project files + reindex
python3 scripts/add-tags.py ~/workspace/memory/projects/
openclaw memory index --force
```

## What's Inside

- **Config patches** — Optimized search parameters for small embedding models
- **Custom memoryFlush prompt** — Auto-generates tags + Chinese word spacing
- **Diagnostic script** — One-click health check for your memory system
- **Maintenance scripts** — Log compression, tag generation, garbage cleanup
- **Templates** — P0/P1/P2 memory architecture, project files, lesson files
- **Weekly cron job** — Automated maintenance (compress, clean, retag, reindex)

## The Technical Details

See [docs/fts5-unicode61-bug.md](docs/fts5-unicode61-bug.md) for the full analysis of the FTS5 CJK tokenization issue.

## Compatibility

- ✅ OpenClaw 2026.3+ with Ollama
- ✅ Small models: qwen3-embedding:0.6b, nomic-embed-text, mxbai-embed-large
- ✅ Chinese-dominant workflows
- ⚠️ Cloud models (OpenAI/Voyage): file structure applies, but parameters need adjustment

## License

[MIT](LICENSE)

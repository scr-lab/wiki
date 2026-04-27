+++
title = "ローカルAiエージェント導入手順書"
description = "ローカル環境でのコーディングエージェントのセットアップについてまとめます．"
date = 2026-04-28
updated = 2026-04-28
authors = ["Yamabe"]
tags = ["Ollama","LLM","Coding"]
category = "knowledge"
draft = false
weight = 1
+++

# Open Code + Ollama でローカル環境でコーディングエージェントを使い倒す．

> **参考記事:** [OpenCode（opencode）から LM Studio / Ollama に接続してローカルLLM（GLM-4.7 / Qwen3-Coder）を使う完全ガイド【macOS / Windows / Linux】](https://note.com/zephel01/n/ndf224d5b6d9a)

## はじめに

研究室でのコーディングを普段ご利用の生成AI（[Claude.ai](http://Claude.ai) など）に任せるのは，研究内容を発表前に外部に漏らすことと同義であり，危険です．
しかしながら，昨今のAIのコーディング力は目を見張るものがあり，活用しないという手もないでしょう．

そこで，本記事では，ローカルLLMである **Ollama** を使用した，安全なコーディングエージェントの導入方法を紹介します．

---

## アーキテクチャ

本構成では，すべての推論がローカルマシン上で完結します．外部APIへのリクエストは一切発生しません．

```
┌─────────────────────────────────────────────────────┐
　                    ローカルマシン                      　
　                                                     　
　   ┌─────────────┐        ┌─────────────────────┐   　
　   　  OpenCode   　 ──────▶　  Ollama API Server  　   　
　   　  (TUI/CLI)  　        　  localhost:11434     　   　
　   └─────────────┘        └──────────┬──────────┘   　
　         　                           　               　
　   [chat agent]              ┌───────┴────────┐      　
　   設計・要約・レビュー         　  LLMモデル群   　      　
　                             　  GLM-4.7       　      　
　   [build agent]             　  Qwen3-Coder   　      　
　   実装・修正・解析            └───────────────┘      　
　                                                     　
　   ※ インターネット接続不要・データ外部流出なし           　
└─────────────────────────────────────────────────────┘
```

- **OpenCode**: TUIベースのコーディングエージェント
- **Ollama**: ローカルLLMを `http://localhost:11434/v1` で OpenAI互換APIとして公開
- **エージェント分離**: チャット用（read-only）と実装用（read/write/bash）でモデルと権限を分割

---

## セットアップ

### 1. Open Code のインストール

**macOS**

```bash
brew install opencode
```

**Linux**

```bash
curl -fsSL https://opencode.ai/install | bash
```

**Windows**

公式サイトからインストール: [https://opencode.ai/download](https://opencode.ai/download)

---

### 2. モデルの選定

**推奨構成**

「要約・設計・相談」と「実装・修正・解析」などに役割を分けてモデルを２つ選ぶとよいです．

| 役割 | 用途 | 推奨モデル例 | 特徴 |
|------|------|-------------|------|
| **chat** | 設計・要約・レビュー | `GLM-4.7-flash-neo-code-max:q4_k_m` | 高速・低VRAM・汎用 |
| **build** | 実装・修正・解析 | `Qwen3-Coder-next:q3_k_m` | コーディング特化・精度高 |

VRAM の目安：

- **8GB**: Q3〜Q4 の 7B〜9Bモデルまで
- **16GB**: Q4 の 14B〜20Bモデル，または上記2モデル同時ロード可能
- **24GB以上**: 30B前後のモデルも快適動作

モデルのダウンロードは以下で行います：

```bash
ollama pull glm-4.7-flash-neo-code-max:q4_k_m
ollama pull qwen3-coder-next:q3_k_m
```

---

### 3. Ollama セットアップ

Ollama をインストール後，サーバーを起動します：

```bash
ollama serve
```

正常起動の確認（別ターミナルで）：

```bash
curl http://localhost:11434/api/tags
```

モデル一覧が JSON で返ってくれば成功です．

---

### 4. opencode.json の設定

プロジェクトルート（または `~/.config/opencode/`）に `opencode.json` を配置します：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "ollama/glm47",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": {
        "baseURL": "http://localhost:11434/v1"
      },
      "models": {
        "glm47": {
          "name": "glm-4.7-flash-neo-code-max:q4_k_m",
          "tools": true
        },
        "qwen3": {
          "name": "qwen3-coder-next:q3_k_m",
          "tools": true
        }
      }
    }
  },
  "agent": {
    "chat": {
      "description": "設計・要約・レビュー（GLM）",
      "model": "ollama/glm-4.7-flash-neo-code-max:q4_k_m",
      "tools": {
        "read": true,
        "write": false,
        "edit": false,
        "bash": false,
        "websearch": false,
        "webfetch": false
      }
    },
    "build": {
      "description": "実装・修正（Qwen Coder）",
      "model": "ollama/qwen3-coder-next:q3_k_m",
      "tools": {
        "read": true,
        "write": true,
        "edit": true,
        "bash": true,
        "websearch": false,
        "webfetch": false
      }
    }
  }
}
```

---

## 5. 起動

```bash
# プロジェクトディレクトリで起動
cd /path/to/your/project
opencode
```

起動後の基本操作：

| キー操作 | 動作 |
|---------|------|
| `Tab` | エージェント切り替え（chat ↔ build） |
| `Ctrl+C` | 処理中断 |
| `Ctrl+D` / `exit` | 終了 |
| `/help` | ヘルプ表示 |

エージェントを明示的に指定して起動する場合：

```bash
opencode --agent chat   # 設計・要約モード
opencode --agent build  # 実装・修正モード
```

---

## 6. トラブルシューティング

### Ollama に接続できない

```
Error: connect ECONNREFUSED 127.0.0.1:11434
```

`ollama serve` が起動しているか確認してください．macOS では `ollama` アプリをメニューバーから起動することもできます．

### モデルが見つからない

```
Error: model "qwen3-coder-next:q3_k_m" not found
```

`ollama pull <モデル名>` でモデルを事前にダウンロードしてください．`ollama list` で現在のモデル一覧を確認できます．

### 応答が極端に遅い

- GPU が使用されているか確認：`ollama ps` でメモリ使用状況を確認
- VRAM 不足の場合，より小さい量子化モデル（`q3_k_m` → `q2_k` など）を試す
- CPU のみの場合，応答速度は大幅に低下します（7B モデルで 10〜20 tok/s 程度）

### `tools: true` でツール呼び出しが失敗する

一部のモデルは function calling に対応していません．`opencode.json` の `"tools": false` に変更するか，ツール対応モデル（Qwen3-Coder, GLM-4 シリーズ等）を使用してください．

### Windows で文字化けが発生する

PowerShell / コマンドプロンプトの文字コードを UTF-8 に設定してください：

```powershell
chcp 65001
```

---

## 7. GitHub

セットアップ用スクリプト（`.ps1` / `.sh`），`opencode.json` テンプレート，プロンプトサンプル集を以下のリポジトリで公開しています：

> 🔗 [https://github.com/your-lab/local-ai-agent-setup](https://github.com/your-lab/local-ai-agent-setup)（← 実際のURLに変更してください）

リポジトリの内容：

- `setup.ps1` — Windows 用ワンライナーセットアップスクリプト
- `setup.sh` — macOS / Linux 用セットアップスクリプト
- `opencode.json` — 設定テンプレート（GLM + Qwen3 構成）
- `prompts/` — コードレビュー・設計・ドキュメント生成などのプロンプトサンプル集

---

## 8. セキュリティ注意点

- `0.0.0.0` で API を公開しない（デフォルトの `localhost` のまま使用する）
- LAN 内でも不用意に晒さない
- 外部からのアクセスが必要な場合は **Cloudflare Access** または **SSH トンネル** を推奨
- `build` エージェントの `bash: true` は強力な権限です．信頼できるプロジェクトでのみ使用し，本番環境では必ず `false` に設定する

---

## 9. モデル選定のアドバイス

**基本方針**：用途・VRAM・速度のバランスで選ぶ

```
精度重視  ◀────────────────────────────▶  速度重視
  Qwen3-Coder 14B         GLM-4.7-flash
  （実装・修正向き）       （相談・要約向き）
```

**チェックリスト**

- [ ] VRAM 8GB 以下 → Q3/Q4 量子化の 7B モデルを選ぶ
- [ ] コード生成メイン → Qwen3-Coder / DeepSeek-Coder 系
- [ ] 日本語での指示が多い → GLM-4 系・Qwen 系は日本語対応が比較的良好
- [ ] ツール呼び出し（function calling）が必要 → `tools: true` 対応モデルを確認
- [ ] 複数モデルを同時ロードしたい → VRAM 16GB 以上推奨

**参考：モデル名の量子化表記の見方**

| 表記 | 意味 | 品質 | サイズ |
|------|------|------|--------|
| `q8_0` | 8-bit 量子化 | ★★★★★ | 大 |
| `q4_k_m` | 4-bit（K-Quant Medium） | ★★★★☆ | 中 |
| `q3_k_m` | 3-bit（K-Quant Medium） | ★★★☆☆ | 小 |
| `q2_k` | 2-bit（K-Quant） | ★★☆☆☆ | 最小 |

VRAM に余裕があれば `q4_k_m` 以上を推奨します．`q3_k_m` は容量節約が優先される場合に使用してください．

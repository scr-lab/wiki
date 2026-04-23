+++
title = "Open Notebook + Ollama でローカル論文処理環境を構築する"
description = "NotebookLM ライクなローカル LLM 環境を Windows PC に構築し、論文PDFの要約・検索・arXiv自動取り込みまでを整備するガイド"
date = 2026-04-23
updated = 2026-04-23
authors = ["Yamabe"]
tags = ["AI", "LLM", "Ollama", "OpenNotebook", "Docker", "research"]
category = "knowledge"
draft = false
weight = 1
[extra]
toc = true
+++

> **参考記事:** [NotebookLMの代替に？Open Notebook + Ollama でローカルLLM環境を構築する (Zenn)](https://zenn.dev/roswell/articles/open-notebook-setup)

## はじめに

Google NotebookLM は研究用途に非常に便利ですが、機密情報・未発表論文・個人データを外部サービスへ送信することには注意が必要です。  
本記事では **Open Notebook** と **Ollama** を組み合わせ、**Windows PC 上で完全ローカルに動作する論文処理環境**を構築します。  
セットアップ手順に加え、論文向けプロンプトテンプレートと **arXiv からの自動取り込みスクリプト**も整備します。

### 全体アーキテクチャ

```
論文PDF / arXiv URL
       │
       ▼
 Open Notebook  ←──────────────────────────────┐
 (Streamlit UI)                                │
       │  RAG / チャット / 要約                  │
       ▼                                       │
   Ollama API  (localhost:11434)               │
  ┌────────────────────────────┐               │
  │ Chat Model    : llama3.1   │  Docker       │
  │ Embed Model   : nomic-embed│  Compose      │
  └────────────────────────────┘               │
       │                                       │
       ▼                                       │
  SurrealDB  (localhost:8000)  ───────────────-┘
  (ベクトル DB + メタデータ)

  arXiv Fetcher Script (Python / cron)
  → PDF を自動ダウンロード → Open Notebook API へ投入
```

---

## 推奨スペック

| 項目 | 推奨 | 最低限 |
|------|------|--------|
| GPU VRAM | 12 GB (RTX 3060 Ti 以上) | 8 GB |
| RAM | 32 GB | 16 GB |
| ストレージ | NVMe 50 GB 以上の空き | HDD 可（低速） |
| OS | Windows 11 64-bit | Windows 10 22H2 以降 |

> VRAM が不足する場合は量子化モデル (q4_K_M) を選択してください（後述）。

---

## Step 1: Ollama のインストール

### 1-1. ダウンロードとインストール

[https://ollama.com/download](https://ollama.com/download) から **Windows 版インストーラー**をダウンロードして実行します。  
インストール後、タスクトレイに Ollama アイコンが表示されれば起動済みです。

### 1-2. モデルのダウンロード

PowerShell を管理者権限で開き、以下を実行します。

```powershell
# ── Chat モデル (VRAM に合わせて選択) ──────────────────────────────
# VRAM 8 GB  → 8B モデル
ollama pull llama3.1:8b

# VRAM 12 GB → 12〜14B モデル（読解力が高い）
ollama pull gemma3:12b

# 多言語・日本語対応重視
ollama pull qwen2.5:14b

# メモリ不足時の量子化版
ollama pull llama3.1:8b-instruct-q4_K_M

# ── Embedding モデル（必須）──────────────────────────────────────
ollama pull nomic-embed-text

# ── インストール確認 ─────────────────────────────────────────────
ollama list
```

> **VRAM 目安:** 8B モデル ≈ 5 GB、12B ≈ 8 GB、14B ≈ 9 GB (q4 量子化時)

---

## Step 2: Docker Desktop のインストール

Open Notebook は Docker Compose で起動します。

1. [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/) から **Docker Desktop for Windows** をインストール。
2. インストール後に再起動し、タスクトレイのアイコンが緑になっていることを確認。
3. 動作確認:

```powershell
docker run hello-world
# → "Hello from Docker!" が表示されれば OK
```

---

## Step 3: Open Notebook のセットアップ

### 3-1. 作業ディレクトリの作成と設定ファイルの取得

```powershell
mkdir $HOME\open-notebook
cd $HOME\open-notebook

# docker-compose.yml を取得
curl -o docker-compose.yml https://raw.githubusercontent.com/lfnovo/open-notebook/main/docker-compose.yml
```

### 3-2. 暗号化キーの生成

```powershell
# PowerShell で 32 バイトのランダム文字列を生成
-join ((1..64) | ForEach-Object { '{0:x}' -f (Get-Random -Max 16) })
# → 生成された文字列をコピーしておく
```

### 3-3. docker-compose.yml の編集

`docker-compose.yml` をテキストエディタで開き、以下の 2 点を修正します。

```yaml
# 変更前
- OPEN_NOTEBOOK_ENCRYPTION_KEY=change-me-to-a-secret-string

# 変更後（先ほど生成したキーを貼り付け）
- OPEN_NOTEBOOK_ENCRYPTION_KEY=<生成した64文字の文字列>
```

Ollama 接続設定を `environment:` ブロックに**追加**します。

```yaml
environment:
  - OPEN_NOTEBOOK_ENCRYPTION_KEY=<your-key>
  - SURREAL_URL=ws://surrealdb:8000/rpc
  - SURREAL_USER=root
  - SURREAL_PASSWORD=root
  - SURREAL_NAMESPACE=open_notebook
  - SURREAL_DATABASE=open_notebook
  # ↓ この行を追加 (Windows / Mac 共通)
  - OLLAMA_BASE_URL=http://host.docker.internal:11434
```

### 3-4. サービス起動

```powershell
cd $HOME\open-notebook
docker compose up -d

# ログ確認 (UTC 表示なので +9h が日本時間)
docker compose logs open_notebook
```

ブラウザで **http://localhost:8502** を開きます。

---

## Step 4: Open Notebook の初期設定

### 4-1. AI プロバイダーの登録

左ペイン → **モデル** → Ollama の **「+ 設定を追加」**

| 項目 | 値 |
|------|----|
| 設定名 | `Ollama Local`（任意） |
| API キー | 不要（空欄） |
| ベース URL | `http://host.docker.internal:11434` |

「コンセントマーク」で接続チェックを実行し、緑のチェックが入れば成功です。

### 4-2. Language / Embedding モデルの割り当て

| 用途 | 推奨モデル | 軽量代替 |
|------|-----------|---------|
| Language (Chat) | `llama3.1:8b` / `gemma3:12b` | `gemma3:1b` |
| Embedding | `nomic-embed-text:latest` | ─ |
| Transformation | Language と同じモデル | ─ |

---

## Step 5: 論文 PDF の取り込みフロー

1. 左ペイン → **新規** → ファイルアップロードダイアログ
2. PDF をドラッグ＆ドロップ（アプリケーション生成 PDF 推奨; スキャン PDF は OCR が必要）
3. **「Embedded 済み: はい」** になるまで待つ（モデルサイズにより数十秒〜数分）
4. ソースをクリックしてチャット開始

> **注意:** スキャン PDF（OCR なし）は Embedding が `Processing...` のまま止まる場合があります。  
> その場合は [OCRmyPDF](https://github.com/ocrmypdf/OCRmyPDF) などで前処理を行ってください。

---

## Step 6: 論文処理プロンプトテンプレート集

Open Notebook のチャット欄にそのままコピペして使えます。  
**日本語で応答させるには、プロンプトの冒頭に必ず「日本語で、」を付けてください。**（モデルによっては英語で返答する場合があります）

### 6-1. 基本要約

```
日本語で、この論文を以下の構成で要約してください。

1. **問題設定**: 何を解こうとしているか
2. **提案手法**: どのようなアプローチか（数式・アルゴリズムの概要も含む）
3. **主要な結果**: ベースラインとの定量比較を中心に
4. **貢献・新規性**: 先行研究と比べて何が新しいか
5. **限界・課題**: 著者が認めている制約や将来課題
```

### 6-2. 手法の詳細理解

```
日本語で、提案手法のアーキテクチャ・アルゴリズムを詳しく説明してください。
以下の点を含めること:
- 入力・出力の形式
- 主要コンポーネントとその役割
- 学習手順（損失関数、最適化）
- 推論時の手順
初学者でも理解できるよう、具体例や直感的な説明を添えてください。
```

### 6-3. 関連研究との比較

```
日本語で、この論文が引用している先行研究のうち最も重要なものを 5 件挙げ、
本論文の手法とそれぞれを以下の観点で比較してください。
- 手法の違い
- 性能差（記載がある場合）
- 本論文が先行研究のどの限界を克服しているか
```

### 6-4. 再現実装チェックリスト

```
日本語で、この論文の手法を再現実装するために必要な情報を整理してください。

- データセット（名前・入手先・前処理）
- モデル構造（レイヤー数、次元数、活性化関数など）
- ハイパーパラメータ（学習率、バッチサイズ、エポック数など）
- 評価指標と評価プロトコル
- 公開コード・チェックポイントのリンク（記載があれば）
- 論文中で不明確な点や再現困難な箇所
```

### 6-5. 批判的レビュー

```
日本語で、この論文を査読者の立場で批判的に評価してください。

**強み:**
- 理論的・実験的に説得力のある点

**弱み・疑問点:**
- 実験設計の問題（比較対象、評価指標の妥当性）
- 主張と証拠の乖離
- 再現性の懸念

**改善提案:**
- 追加すべき実験・分析
```

### 6-6. スライド用サマリー生成

```
日本語で、この論文を研究室ゼミで発表するための箇条書きサマリーを作成してください。
スライド 1 枚につき 3〜5 箇条書き、全体で 5 スライド分（導入・手法・実験・結果・まとめ）。
専門用語には簡単な説明を括弧で補足してください。
```

### 6-7. 複数論文の横断比較（複数ソースをノートブックに追加した後）

```
日本語で、このノートブックに登録されているすべての論文を横断的に比較してください。

比較軸:
- タスク・ドメイン
- 手法カテゴリ
- 使用データセット
- 主要評価指標と最高スコア
- 計算コスト（パラメータ数・推論速度の記載があれば）

表形式でまとめ、その後に総評を加えてください。
```

---

## Step 7: arXiv 自動取り込みスクリプト

### 7-1. 依存パッケージのインストール

```powershell
pip install arxiv requests schedule
```

### 7-2. 取り込みスクリプト本体

`arxiv_fetcher.py` として保存します。

```python
"""
arxiv_fetcher.py
────────────────────────────────────────────────────────────────────────────
arXiv から指定キーワード・カテゴリの最新論文を取得し、
Open Notebook の REST API 経由で自動登録するスクリプト。

使い方:
  python arxiv_fetcher.py                      # 1 回実行
  python arxiv_fetcher.py --schedule           # 毎日 8:00 に定期実行
  python arxiv_fetcher.py --dry-run            # 取得のみ（Open Notebook へは送信しない）
"""

import arxiv
import requests
import json
import argparse
import schedule
import time
import logging
from datetime import datetime, timedelta, timezone
from pathlib import Path

# ────────────────────────── 設定 ──────────────────────────

# 検索クエリ（arXiv の検索構文が使えます）
SEARCH_QUERIES = [
    "ti:diffusion model AND cat:cs.CV",          # CS.CV: 拡散モデル
    "ti:large language model AND cat:cs.CL",     # CS.CL: LLM
    "abs:retrieval augmented generation",        # RAG 全カテゴリ
    "ti:protein structure AND cat:q-bio.BM",     # 生物情報
]

# 1 クエリあたりの最大取得件数
MAX_RESULTS_PER_QUERY = 5

# 過去何日分の論文を取得するか（重複を防ぐため短くする）
SINCE_DAYS = 1

# Open Notebook API エンドポイント
OPEN_NOTEBOOK_URL = "http://localhost:8502"

# ダウンロードした PDF の保存先
PDF_DIR = Path.home() / "open-notebook" / "arxiv_pdfs"

# 取得済み arXiv ID を記録するファイル（重複防止）
SEEN_IDS_FILE = Path.home() / "open-notebook" / ".seen_arxiv_ids.json"

# ログ設定
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler(Path.home() / "open-notebook" / "arxiv_fetcher.log"),
        logging.StreamHandler(),
    ],
)
log = logging.getLogger(__name__)

# ─────────────────────────────────────────────────────────

def load_seen_ids() -> set:
    if SEEN_IDS_FILE.exists():
        with open(SEEN_IDS_FILE) as f:
            return set(json.load(f))
    return set()

def save_seen_ids(ids: set):
    SEEN_IDS_FILE.parent.mkdir(parents=True, exist_ok=True)
    with open(SEEN_IDS_FILE, "w") as f:
        json.dump(list(ids), f, indent=2)

def fetch_papers(query: str, max_results: int, since_days: int) -> list[arxiv.Result]:
    """arXiv から論文を取得する"""
    client = arxiv.Client()
    search = arxiv.Search(
        query=query,
        max_results=max_results,
        sort_by=arxiv.SortCriterion.SubmittedDate,
        sort_order=arxiv.SortOrder.Descending,
    )
    cutoff = datetime.now(timezone.utc) - timedelta(days=since_days)
    results = []
    for paper in client.results(search):
        if paper.published and paper.published < cutoff:
            break
        results.append(paper)
    return results

def download_pdf(paper: arxiv.Result, dest_dir: Path) -> Path | None:
    """PDF をダウンロードして保存パスを返す"""
    dest_dir.mkdir(parents=True, exist_ok=True)
    arxiv_id = paper.entry_id.split("/")[-1]
    pdf_path = dest_dir / f"{arxiv_id}.pdf"
    if pdf_path.exists():
        log.info(f"  [SKIP] PDF already exists: {pdf_path.name}")
        return pdf_path
    try:
        paper.download_pdf(dirpath=str(dest_dir), filename=f"{arxiv_id}.pdf")
        log.info(f"  [DL]   {pdf_path.name}")
        return pdf_path
    except Exception as e:
        log.error(f"  [ERR]  PDF download failed for {arxiv_id}: {e}")
        return None

def register_to_open_notebook(paper: arxiv.Result, pdf_path: Path) -> bool:
    """
    Open Notebook の /api/source エンドポイントへ PDF を登録する。
    Open Notebook の API 仕様に合わせてエンドポイントを調整してください。
    """
    url = f"{OPEN_NOTEBOOK_URL}/api/source"
    try:
        with open(pdf_path, "rb") as f:
            resp = requests.post(
                url,
                files={"file": (pdf_path.name, f, "application/pdf")},
                data={
                    "title": paper.title,
                    "description": paper.summary[:500],
                    "tags": ",".join(paper.categories),
                },
                timeout=60,
            )
        if resp.status_code in (200, 201):
            log.info(f"  [OK]   Registered: {paper.title[:60]}")
            return True
        else:
            log.warning(f"  [WARN] API returned {resp.status_code}: {resp.text[:200]}")
            return False
    except requests.exceptions.ConnectionError:
        log.error(
            "  [ERR]  Cannot connect to Open Notebook. "
            "Is the Docker container running? (docker compose up -d)"
        )
        return False
    except Exception as e:
        log.error(f"  [ERR]  Registration failed: {e}")
        return False

def run(dry_run: bool = False):
    log.info("=" * 60)
    log.info(f"arXiv fetch started at {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    log.info("=" * 60)

    seen_ids = load_seen_ids()
    new_ids = set()

    for query in SEARCH_QUERIES:
        log.info(f"\n[QUERY] {query}")
        papers = fetch_papers(query, MAX_RESULTS_PER_QUERY, SINCE_DAYS)
        log.info(f"  → {len(papers)} paper(s) found")

        for paper in papers:
            arxiv_id = paper.entry_id.split("/")[-1]

            if arxiv_id in seen_ids:
                log.info(f"  [SKIP] Already processed: {arxiv_id}")
                continue

            log.info(f"  [NEW]  {arxiv_id} | {paper.title[:60]}")

            if dry_run:
                log.info("  [DRY]  Skipping download & registration (--dry-run)")
                new_ids.add(arxiv_id)
                continue

            pdf_path = download_pdf(paper, PDF_DIR)
            if pdf_path:
                success = register_to_open_notebook(paper, pdf_path)
                if success:
                    new_ids.add(arxiv_id)

    seen_ids |= new_ids
    save_seen_ids(seen_ids)
    log.info(f"\n✓ Done. {len(new_ids)} new paper(s) processed.")

def main():
    parser = argparse.ArgumentParser(description="arXiv → Open Notebook 自動取り込み")
    parser.add_argument("--schedule", action="store_true", help="毎日 08:00 に定期実行")
    parser.add_argument("--dry-run", action="store_true", help="取得のみ（登録しない）")
    args = parser.parse_args()

    if args.schedule:
        log.info("Scheduler started. Will run daily at 08:00.")
        schedule.every().day.at("08:00").do(run, dry_run=args.dry_run)
        run(dry_run=args.dry_run)   # 起動時に即時実行
        while True:
            schedule.run_pending()
            time.sleep(60)
    else:
        run(dry_run=args.dry_run)

if __name__ == "__main__":
    main()
```

### 7-3. 使い方

```powershell
# 1 回だけ実行（動作確認）
python arxiv_fetcher.py --dry-run   # ダウンロードのみ、Open Notebook へは送信しない
python arxiv_fetcher.py             # 実際に登録

# 毎日 08:00 に自動実行（バックグラウンドで常駐）
python arxiv_fetcher.py --schedule
```

### 7-4. 検索クエリのカスタマイズ例

```python
SEARCH_QUERIES = [
    # 特定著者の論文
    "au:Vaswani_A",

    # 複数カテゴリ
    "ti:graph neural network AND (cat:cs.LG OR cat:cs.AI)",

    # 特定学会の arXiv プレプリント
    "ti:NeurIPS 2025",

    # 日本語 NLP 系
    "abs:Japanese language model",

    # 材料科学 × 機械学習
    "ti:machine learning AND cat:cond-mat.mtrl-sci",
]
```

### 7-5. Windows タスクスケジューラへの登録（常駐不要の場合）

`--schedule` フラグの代わりに、Windows タスクスケジューラで毎朝実行することもできます。

```powershell
# タスクスケジューラに登録（毎日 08:00 実行）
$action  = New-ScheduledTaskAction -Execute "python" `
           -Argument "$HOME\arxiv_fetcher.py" `
           -WorkingDirectory "$HOME\open-notebook"
$trigger = New-ScheduledTaskTrigger -Daily -At "08:00"
Register-ScheduledTask -TaskName "arXivFetcher" `
    -Action $action -Trigger $trigger -RunLevel Highest
```

---

## トラブルシューティング

### Ollama が応答しない

```powershell
# タスクトレイのアイコンがない場合は手動起動
ollama serve

# API の疎通確認
curl http://localhost:11434/api/tags
```

### Open Notebook の画面が真っ白

Docker コンテナを再起動してください。

```powershell
cd $HOME\open-notebook
docker compose restart
```

解消しない場合は PC を再起動すると改善する事例があります（Zenn 記事参照）。

### チャットの応答が返ってこない / 固まる

大きなモデル（7B 以上）では推論に 30 秒以上かかることがあります。  
ブラウザをリロードすると、バックグラウンドで生成されていた応答が表示されます。  
根本解決には軽量モデル（`gemma3:1b` など）への変更が効果的です。

### PDF の Embedding が「Processing...」のまま

スキャン PDF（画像 PDF）は OCR なしでは処理できません。  
以下のコマンドで OCR を適用してから再アップロードしてください。

```powershell
pip install ocrmypdf
ocrmypdf input.pdf output_ocr.pdf --language jpn+eng
```

### VRAM 不足でモデルが落ちる

```powershell
# 量子化モデルに切り替える（VRAM 使用量が約 40% 削減）
ollama pull llama3.1:8b-instruct-q4_K_M
```

`docker-compose.yml` の `DEFAULT_CHAT_MODEL` をこのモデル名に合わせて変更します。

---

## まとめ

| 機能 | 実現方法 |
|------|---------|
| ローカル LLM チャット | Ollama + Open Notebook |
| 論文 PDF の RAG 検索 | nomic-embed-text + SurrealDB |
| 論文要約・批評 | プロンプトテンプレート集（Step 6） |
| arXiv 自動取り込み | `arxiv_fetcher.py`（Step 7） |
| 定期実行 | Windows タスクスケジューラ / `--schedule` フラグ |

プライバシーを保ちながら、自分の研究テーマに特化した論文データベースを育てていくことができます。  
モデルの選択やプロンプトは研究分野に合わせて自由にカスタマイズしてください。

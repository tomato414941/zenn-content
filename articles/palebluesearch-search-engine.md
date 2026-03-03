---
title: "個人で Web 検索エンジンを作って AI エージェント向け API にした話"
emoji: "🔍"
type: "tech"
topics: ["python", "検索エンジン", "AI", "MCP", "FastAPI"]
published: false
---

## はじめに

AI エージェントが Web 検索する時代になった。Claude や GPT にツールを渡せば、エージェントが自律的に検索して情報を集めてくれる。

ただ、既存の検索 API を使っていて2つ気になることがあった。

1. **検索結果がいつの情報なのかわからない**。AI エージェントが最新情報を求めているのに、何年も前の記事を根拠にしていたら意味がない。
2. **低品質なページが混ざる**。広告だらけのアグリゲーションサイトや、中身のないボイラープレートページが検索結果に入ってくる。

そこで自分で検索エンジンを作ることにした。鮮度メタデータとコンテンツ品質スコアリングを備えた、AI エージェント向けの検索 API。名前は **PaleBlueSearch**。

現在 68万ページ以上をインデックスし、無料 API として公開している。

```bash
curl "https://palebluesearch.com/api/v1/search?q=python+web+framework"
```

```json
{
  "query": "python web framework",
  "total": 42,
  "hits": [
    {
      "url": "https://example.com/fastapi",
      "title": "FastAPI - Modern Python Web Framework",
      "snip_plain": "A modern, fast web framework for building APIs with Python...",
      "rank": 12.5,
      "indexed_at": "2026-03-01T12:00:00.000000+00:00",
      "published_at": "2026-02-28T09:30:00+00:00"
    }
  ],
  "mode": "auto"
}
```

この記事では、設計・実装・運用で学んだことを共有する。

## アーキテクチャ

CQRS-lite（読み書き分離）のマイクロサービス構成を採用した。

```
                    ┌─────────────┐
                    │   Crawler   │ ← Web をクロール
                    │  (Worker)   │
                    └──────┬──────┘
                           │ URL + HTML
                           ▼
┌──────────────┐    ┌─────────────┐
│   Frontend   │    │   Indexer   │ ← トークン化 + 埋め込み
│ (Search API) │    │ (Write Node)│
│  読み取り専用  │    └──────┬──────┘
└──────┬───────┘           │
       │                   ▼
       │           ┌──────────────┐
       └──────────→│  PostgreSQL  │
                   │  + pgvector  │
                   │  OpenSearch  │
                   └──────────────┘
```

| サービス | 役割 | 技術 |
|---------|------|------|
| Frontend | 検索 UI + API | FastAPI + Gunicorn |
| Indexer | インジェスト + トークン化 + 埋め込み | FastAPI + Uvicorn |
| Crawler | 分散クローラー | aiohttp + trafilatura |
| DB | メインストレージ + ベクトル検索 | PostgreSQL 16 + pgvector |
| 検索エンジン | BM25 全文検索 | OpenSearch 2.17 |

なぜマイクロサービスにしたか？検索（読み取り）とクロール+インデックス（書き込み）は負荷特性がまったく違う。検索は低レイテンシが命、クロールは高スループットが命。分離することで独立にスケールできる。

とはいえ、全サービスが1台の VPS（Hetzner CPX31: 4vCPU, 8GB RAM）で動いている。個人開発のスケール感。

## クローラー: 60万ページへの道

### 基本設計

クローラーは `aiohttp` ベースの非同期ワーカーで、以下のサイクルを回す。

1. DB から pending URL を取得（ドメイン分散）
2. `robots.txt` をチェック
3. HTML をダウンロード
4. リンクを抽出して新しい URL をキューに追加
5. コンテンツを Indexer に送信

並行度は `CRAWL_CONCURRENCY` で制御している。本番では10並行。

### ドメイン多様性

検索結果が1つのドメインに偏らないよう、2つのレイヤーで多様性を確保した。

**クロール時**: ドメインごとの pending URL 上限を設けて、巨大サイトがキューを占拠するのを防ぐ。

**検索時**: 結果を `MAX_PER_DOMAIN`（デフォルト5件）でキャップする。

```python
def diversify_hits(hits, limit, max_per_domain=5):
    seen = defaultdict(int)
    result = []
    for hit in hits:
        domain = _extract_domain(hit.url)
        if seen[domain] < max_per_domain:
            result.append(hit)
            seen[domain] += 1
        if len(result) >= limit:
            break
    return result
```

### ドメインブロックリスト

クロールしても検索価値が低いドメインはブロックリストで除外している。

- **SNS**: twitter.com, instagram.com, tiktok.com（ログイン壁、低テキスト密度）
- **動画**: youtube.com, vimeo.com（テキストとして検索する意味がない）
- **EC**: amazon.com, rakuten.co.jp（膨大な商品ページ、AI 検索には不向き）
- **URL 短縮**: t.co, bit.ly（リダイレクトのみ）

### シードリスト

最初のクロール対象は [Tranco Top 1M](https://tranco-list.eu/) から選んだ。ニュース、技術ブログ、Wikipedia など、テキストコンテンツが充実したドメインを優先。

### コンテンツ品質スコアリング

クロールしたページすべてが検索に値するわけではない。広告だらけのページやリンク集は、AI エージェントにとってノイズでしかない。

そこで、各ページにコンテンツ品質スコアを付与している。

- **テキスト密度**: HTML 全体に対する本文テキストの比率。低ければ広告やナビゲーション主体
- **リンク比率**: テキスト量に対するリンク数。高ければアグリゲーションページ
- **構造シグナル**: 見出し、段落、リストの有無。構造化された記事ほど高スコア

このスコアを検索ランキングに反映し、低品質ページを自動的に降格させている。

また、コンテンツ抽出には [trafilatura](https://trafilatura.readthedocs.io/) を使い、ナビゲーション、フッター、サイドバーを除去して本文だけをインデックスしている。

## 検索の仕組み

3つの検索モードを用意している。

### BM25（キーワード検索）

OpenSearch を使った古典的な全文検索。デフォルトモード。

```python
query = {
    "multi_match": {
        "query": query_tokens,
        "fields": ["title^3", "content"],
        "type": "cross_fields",
    }
}
```

ポイント:
- **title に3倍のブースト**: タイトルにクエリが含まれるページを優先
- **authority ブースト**: PageRank スコアを `field_value_factor` で反映
- **鮮度減衰**: `indexed_at` に exponential decay（半減期180日）を適用

### ベクトル検索（セマンティック）

pgvector + OpenAI の embedding（1536次元）を使ったコサイン類似度検索。

```sql
SELECT url, title, content,
       1 - (embedding <=> %s::vector) AS similarity
FROM page_embeddings
JOIN documents ON documents.url = page_embeddings.url
ORDER BY embedding <=> %s::vector
LIMIT %s OFFSET %s
```

クエリの意味を理解した検索ができる。「Python の Web フレームワーク」で検索すると、"FastAPI" や "Django" が「Python」「フレームワーク」という単語が含まれていなくてもヒットする。

### ハイブリッド検索

BM25 とベクトル検索を組み合わせる。RRF（Reciprocal Rank Fusion）で統合。タイムアウト3秒で、embedding 生成が間に合わなければ BM25 にフォールバック。

### 日本語対応

日本語は単語の区切りがないため、形態素解析が必須。SudachiPy を使っている。

インデックス時にテキストをトークン化し、スペース区切りのトークン列として OpenSearch に投入。検索クエリも同じトークナイザーを通す。これにより日本語でも BM25 が正しく動く。

## Freshness Metadata

すべての検索結果に2つのタイムスタンプを付けている。

| フィールド | 意味 | 取得方法 |
|-----------|------|---------|
| `indexed_at` | クロール時刻 | クロール時に記録 |
| `published_at` | 元記事の公開日 | HTML の `<meta>` タグ、`<time>` 要素、JSON-LD から抽出 |

### なぜこれが重要か

AI エージェントが検索結果を使うとき、**情報の鮮度を判断する手段**が必要になる。

- 「最新の Python 3.13 の変更点」を調べたいのに、Python 3.9 の記事が返ってきても困る
- 法律や規制に関する情報は、改正日以降の記事でないと正確でない
- 技術記事は1年前の情報でも古くなっている可能性がある

`published_at` があれば、エージェントは「2025年以降の記事だけを使う」という判断ができる。`indexed_at` があれば、「このページは最後に1週間前に確認された」とわかる。

コンテンツ品質スコアで「信頼できるページ」を選び、鮮度メタデータで「いつの情報か」を明示する。この2つの組み合わせが PaleBlueSearch の差別化ポイントになっている。

## MCP サーバー: Claude から直接使えるようにする

検索 API を作っただけでは、Claude Code や Claude Desktop から使うには HTTP リクエストを手動で組み立てる必要がある。MCP（Model Context Protocol）サーバーを作れば、ネイティブなツールとして統合できる。

### 実装

FastMCP を使って薄いラッパーを書いた。実質100行ほど。

```python
from mcp.server.fastmcp import FastMCP
from paleblue_mcp.client import PaleBlueClient

mcp = FastMCP("PaleBlueSearch")
_client = PaleBlueClient()

@mcp.tool()
async def web_search(query: str, limit: int = 10, mode: str = "auto") -> str:
    """Search the web using PaleBlueSearch."""
    data = await _client.search(query=query, limit=limit, mode=mode)
    return _format_hits(data)  # Markdown 形式で返す
```

HTTP クライアント方式を採用し、フロントエンドの REST API を `httpx` で呼んでいる。DB に直接アクセスしないので、依存関係がシンプル。

### 設定

Claude Code の `.mcp.json` に追加するだけ:

```json
{
  "mcpServers": {
    "paleblue-search": {
      "command": "python",
      "args": ["-m", "paleblue_mcp"],
      "cwd": "/path/to/web-search/mcp",
      "env": { "PYTHONPATH": "src" }
    }
  }
}
```

これで Claude に「PaleBlueSearch で Python の Web フレームワークを検索して」と言えば、`web_search` ツールが呼ばれて結果が返ってくる。

## 運用の現実

### インフラ

全サービスが Hetzner の VPS 1台で動いている。

- **サーバー**: CPX31（4vCPU, 8GB RAM）、シンガポールリージョン
- **デプロイ**: Coolify（セルフホスト PaaS）、Git push で自動デプロイ
- **ブランチ戦略**: `main` → STG（自動）、`production` → PRD（手動昇格）

月額コストはサーバー代のみ。OpenAI の embedding API 代は従量課金だが、バックフィル以外ではほぼゼロ。

### PostgreSQL CPU 83% 事件

ある日、PostgreSQL の CPU 使用率が83%に張り付いた。

原因を調べると、`urls` テーブルの `status` カラム（`pending` → `crawling` → `done`）の更新が、4本の partial index を毎回更新していた。クロール並行度10で115 updates/sec、各更新で4本のインデックスが書き換わる。

解決策: インデックスを再設計した。

- `idx_urls_status_domain` を DROP → `idx_urls_pending_domain WHERE status = 'pending'` に置換
- `idx_urls_recrawl`, `idx_urls_last_crawled` を DROP（キャッシュで保護）
- 結果: `crawling` → `done` 遷移のインデックス操作がゼロになり、HOT update が有効化

CPU 使用率は83%から大幅に改善した。PostgreSQL のインデックス設計は、書き込み負荷に直結するということを身をもって学んだ。

### 未使用インデックスの削除

`pg_stat_user_indexes` で未使用のインデックスを発見し、合計 1.2GB のディスクを解放した。

| インデックス | サイズ | 使用回数 |
|------------|-------|---------|
| idx_links_dst | 889MB | 0 |
| idx_crawl_logs_url | 156MB | 0 |
| idx_urls_pending | 104MB | 0（別のインデックスが代替） |

インデックスは SELECT を速くするが、INSERT/UPDATE を遅くする。読まれないインデックスは純粋なコスト。

## まとめ

個人で検索エンジンを作って学んだこと:

1. **検索エンジンは「クロール」が一番大変**。robots.txt 準拠、レート制限、HTML パース、リンク抽出。コードの半分以上がクローラー関連。
2. **BM25 は今でも強い**。ベクトル検索は意味理解に優れるが、キーワードの完全一致では BM25 が圧倒的。ハイブリッドが最適解。
3. **PostgreSQL のインデックス設計は書き込みパフォーマンスに直結する**。特に頻繁に更新されるカラムの partial index には注意。
4. **AI エージェントに必要なのは「品質 + 鮮度」**。URL とスニペットだけでなく、コンテンツ品質と鮮度情報があると AI の判断精度が上がる。

### 現在の規模

| 指標 | 数値 |
|------|------|
| インデックス済みページ | 68万+ |
| 訪問済み URL | 103万+ |
| クロールキュー | 250万+ URL |
| 検索モード | BM25 / ハイブリッド / セマンティック |
| 日本語対応 | SudachiPy 形態素解析 |
| API | 無料（100 req/min） |

### リンク

- **API**: https://palebluesearch.com/api/v1/search?q=test
- **GitHub**: https://github.com/tomato414941/web-search
- **MCP サーバー**: https://github.com/tomato414941/web-search/tree/main/mcp

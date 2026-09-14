# ndbopendata-hub

第10回・第11回NDB（ナショナルデータベース）オープンデータのうち、特定健診の「質問票」「検査」関連を対象にした個人開発の公開ハブです。

公開サイト: https://ndbopendata-hub.com

本リポジトリに含まれるデータやサンプルは、いずれもNDBオープンデータで公開されている統計上の集計値であり、個人情報を含みません。

— Quick links —
- [更新情報](#release-update)
- [Tursoへのバックエンド移行](#turso-migration)
- [APIエンドポイント](#api-endpoints)
- [外部AIから使う](#external-ai-access)
- [NDBヘルスインサイト](#insights)
- [開発の進行概要](#progress)
- [データ構造](#data-structure)
- [データ処理アーキテクチャ](#processing-arch)
- [データベース構成](#db-schema)
- [開発手法（AI協働）](#dev-method)
- [お問い合わせ](#contact)

<a id="release-update"></a>
## 🆕 更新情報（2026年9月14日）

<a id="turso-migration"></a>
### SupabaseからTursoへバックエンドを移行

2026年9月14日、公開サイトのデータ参照先をSupabase PostgreSQLからTurso（SQLite / libSQL）へ切り替え、Vercelへの本番デプロイと表示・検索の確認を完了しました。既存のSupabase上の集計データを移行しており、新しい公開回の追加やExcelからのデータ再作成は行っていません。

現在の構成は **ブラウザー・外部AI → Next.jsの画面／API／MCP（Vercel）→ Turso** です。データベースへはサーバー側から読取専用で接続し、接続キーをブラウザーや公開リポジトリへ配布しません。公開サイトのURL、REST APIとMCPの登録先、公開回を指定する `release=10|11|latest` は継続しています。

移行に伴い、地域・性別・検査名の扱いを揃え、検索時の索引利用と初回の重複取得を改善しました。切替前には移行対象データの全件比較、代表8条件の照合、API・主要画面・MCPの確認を実施し、切替後は質問票、基本／詳細健診、地域集計、地域比較、スマートフォン表示、MCPの検索と取得を含む軽量テスト9項目を通過しました。

従来のExcel取込時の4点突合はデータ作成の検証方法として残し、今回のDB移行の受入ではデータ比較と代表条件の確認を採用しています。旧Supabase環境は切戻し用に保持しており、停止・削除は今回の切替に含めていません。

移行後のデータの見方は[データ構造の解説](reference/DATA_STRUCTURE.md#turso-migration)を参照してください。

### 第11回データの公開（2026年7月14日）

第11回NDBオープンデータを追加し、公開サイトでは第10回・第11回を切り替えて参照できるようにしました。APIでは `release=10` / `release=11` / `release=latest` を指定できます。

第11回は「令和6年度レセプト・令和5年度特定健診分」に対応します。第10回と同じDB構造に追加登録しており、扱うデータは引き続き個票ではなく、地域・性別・年齢階級・値範囲ごとに集計済みの統計値です。

登録済み集計レコード数の比較（第11回公開対応時点）:

| 対象 | 第10回 | 第11回 |
| --- | ---: | ---: |
| 基本検査結果 | 331,945 | 370,179 |
| 詳細検査結果 | 135,718 | 139,498 |
| 質問票回答 | 278,334 | 278,816 |

第11回の取得対象は、特定健診検査76ファイル、質問票44ファイルです。このうち平均値ファイル4件は第10回と同様にDB投入対象外とし、人数・回答数として表示できる集計データを公開対象にしています。

<a id="api-endpoints"></a>
## 🌐 APIエンドポイント（v1）

| パス | 説明 |
| ---- | ---- |
| `GET /api/v1/capabilities` | 利用可能なデータセット・ディメンション・フォーマットを返す discover API |
| `GET /api/v1/items` | 検査項目マスタ（item_id, item_category, unit） |
| `GET /api/v1/areas?type=prefecture` | 都道府県のコード／名称一覧（`type=secondary_medical_area` で二次医療圏） |
| `GET /api/v1/range-labels?item_name=BMI&record_mode=basic` | レンジラベルを安定ID（`range_id`）付きで取得 |
| `GET /api/v1/inspection-stats?...` | 人数集計。`value_range` に `range_id` を指定すると安定してフィルタ可 |
| `GET /api/v1/health` | 稼働状態確認 |
| `GET /api/v1/version` | データ更新日時・スキーマバージョン |

### 自律探索の最小ステップ

```bash
# 1) レンジラベル発見
curl -s "https://ndbopendata-hub.com/api/v1/range-labels?release=latest&item_name=BMI&record_mode=basic"

# 2) 取得した range_id を使って人数を取得
curl -s "https://ndbopendata-hub.com/api/v1/inspection-stats?release=latest&item_name=BMI&record_mode=basic&area_type=prefecture&prefecture_code=02&gender=M&age_group=40-44&value_range=142"
```

レスポンスにはレート制限ヘッダ（`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`）を付与しています。

<a id="external-ai-access"></a>
## 🤖 外部AIから使う

> [!NOTE]
> **production公開済みです。** 2026-07-14の同一runで、`/mcp` のstable `2025-11-25`、7 tools、任意SQLtool不存在、release-aware REST/OpenAPI、旧任意SQL経路404を確認しました。

公開面は集計データ専用のread-only / no-auth APIです。新規MCPクライアントには、stable MCP `2025-11-25`のStreamable HTTP endpoint `https://ndbopendata-hub.com/mcp` を登録します。旧 `https://ndbopendata-hub.com/api/mcp` は互換aliasで、新規設定には使用しません。

| 利用先 | 接続方式 | 更新後の登録先 |
| --- | --- | --- |
| **claude.ai** | Custom Connector / Remote MCP | `https://ndbopendata-hub.com/mcp` |
| **Claude Desktop** | Custom Connector / Remote MCP | `https://ndbopendata-hub.com/mcp` |
| **Claude Code** | `claude mcp add --transport http ndb-opendata ...` | `https://ndbopendata-hub.com/mcp` |
| **Codex CLI / Desktop / IDE** | `codex mcp add ndb-opendata --url ...` | `https://ndbopendata-hub.com/mcp` |
| **ChatGPT Apps** | Developer mode / Remote MCP | `https://ndbopendata-hub.com/mcp` |
| **OpenAI Responses API** | built-in `mcp` tool | `server_url: https://ndbopendata-hub.com/mcp` |
| **GPT Actions** | REST / OpenAPI（MCPとは別経路） | `https://ndbopendata-hub.com/mcp/openapi` |

各data toolとrelease-aware RESTでは `release=10|11|latest`（既定`latest`）を指定できます。一連の分析では同じ公開回を維持してください。検査recordは物理table間でIDが衝突し得るため、単一recordの`fetch`ではなく`ndb_inspection_search`を使用します。`fetch`は質問票record専用です。

direct PostgreSQL MCP、旧stdio/SSE bridge、利用者指定SQLの`/api/mcp-query`は退役済みです。新しい環境へDB credential、bridge設定、旧URLをコピーしないでください。

画面付きの[自然言語アクセスガイド](https://ndbopendata-hub.com/mcp/guide)もproduction反映済みです。HTTP production smokeは完了していますが、ChatGPT、Claude、Codex等の各製品UIからのlive接続確認は製品ごとに別管理します。

<a id="insights"></a>
## 📈 NDBヘルスインサイト（分析記事）

NDBオープンデータから抽出した地域別・性別・年代別の健康リスク分析を公開しています。

**🌐 インサイト一覧**: https://ndbopendata-hub.com/insights

| カテゴリ | 注目記事 |
|---------|---------|
| 代謝・肥満 | [沖縄男性メタボ複合リスク](https://ndbopendata-hub.com/insights/okinawa-metabolic-hotspot/male-prefecture) |
| 腎機能 | [君津男性 内臓脂肪×腎リスク](https://ndbopendata-hub.com/insights/kimitsu-visceral-ckd/male-secondary) |
| 糖尿病 | [高崎・安中 糖腎ダブルリスク](https://ndbopendata-hub.com/insights/takasaki-diabetes-ckd/male-secondary) |
| 血圧 | [最上男性 重度高血圧ギャップ](https://ndbopendata-hub.com/insights/mogami-hypertension-awareness/male-secondary) |
| 女性健康 | [山形女性 ロコモ兆候](https://ndbopendata-hub.com/insights/yamagata-female-mobility/female-prefecture) |
| 生活習慣 | [青森男性 運動不足×血糖](https://ndbopendata-hub.com/insights/aomori-exercise-glucose/male-prefecture) |

> 本分析はAIがNDBオープンデータを活用して試験的に抽出したインサイトです。

## 📊 対象データセット

| 公開回 | 表示ラベル | DB上の識別 |
| --- | --- | --- |
| 第10回 | 令和5年度レセプト・令和4年度特定健診分 | `release=10`, `data_year=2023` |
| 第11回 | 令和6年度レセプト・令和5年度特定健診分 | `release=11`, `data_year=2024` |

データ内容:
- 特定健診質問票（22項目）
- 特定健診検査データ（基本情報レコード・詳細情報レコード）
- 都道府県・二次医療圏、性別、年齢階級、値範囲ごとの集計値

<a id="data-structure"></a>
## 📚 重要：データ構造について

本プロジェクトが扱うデータは、個票（個人単位）ではなく「集計済み統計データ」です。例えば「地域×性別×年齢層×値範囲＝人数」のような1つの集計ポイントを1レコードとして格納・提供します。設計方針や用語の使い分け、4点突合（Excel/DB/API/UI）の整合基準など、詳細は下記の解説をご参照ください。

- データ構造の解説: ./reference/DATA_STRUCTURE.md

<a id="processing-arch"></a>
## 🧰 データ処理アーキテクチャ

本プロジェクトでは、ファイル固有の構造差や例外に堅牢に対応するため、Excelを「1ファイル＝1スクリプト」で読み取る方式を採用しました。

- 戦略: 1ファイル1リーダー（118ファイルを個別に処理）
- 品質保証: テスト駆動（TDD）で各リーダーに標準テストを付与
- 主な利点: エラー分離・デバッグ容易・保守性向上・品質の局所改善が可能
- 解説ドキュメント: ./reference/README_excel_readers.md

<a id="db-schema"></a>
## 🗄️ データベース構成（論理）

公開対象の主な論理スキーマは次のとおりです（名称は正規化済み）。

以下は元データと取込処理を含む論理構成です。公開サイトはTurso上の参照用テーブルを利用するため、物理テーブル名や配置と1対1で対応するものではありません。内部インポート履歴は公開対象に含めません。

### 特定健診質問票系
- `questionnaire_questions`: 質問項目マスタ（22問）
- `questionnaire_answer_options`: 回答選択肢マスタ
- `questionnaire_responses`: 回答データ（都道府県別・二次医療圏別）
- `questionnaire_import_history`: 内部インポート履歴（公開API/MCPから非公開）

### 特定健診検査データ系
- `health_inspection_items`: 検査項目マスタ（27項目）
- `health_inspection_value_ranges`: 検査値階層マスタ（性別対応を含む）
- `basic_checkup_results`: 基本情報レコードの検査結果（地域・性別・年齢層・値範囲ごとの集計）
- `detailed_checkup_results`: 詳細情報レコードの検査結果（医師判定ありの層）
- `health_inspection_import_history`: 内部インポート履歴（公開API/MCPから非公開）

### 地域マスタ
- `prefectures`: 都道府県マスタ
- `secondary_medical_areas`: 二次医療圏マスタ

<a id="progress"></a>
## 🔄 開発の進行概要（2026年9月14日）

- 2025-06（初期）
  - 仮のWebページ開発（プロトタイプUI／試験的API）。NDB第10回データの基本設計・DB正規化・初期の参照用画面を短期で構築。
  - MCPもプロトタイプ（PostgreSQL MCP Server と LLM 接続の試行）。
- 2025-07（基盤強化）
  - Excel個別リーダー方式へ全面転換（1ファイル=1スクリプト）。TDD確立、4点突合（Excel/DB/API/UI）の品質ゲートを導入。
- 2025-09（公開用リニューアル）
  - Webを公開前提で再設計（UI整理、公開ドキュメント整備、配布戦略の明文化）。
  - MCPをHTTP/OpenAPIで外部公開可能な形に拡張（Actions/Apps/Connectors導線・レート制限・CORS等）。
- 2026-07（第11回データ対応）
  - 第11回NDBオープンデータを追加し、第10回・第11回・latestを切り替えられる構成へ更新。
  - 第10回と同じ論理スキーマで第11回の検査・質問票データを追加し、公開回の取り違えを避けるためAPI/UIをrelease-aware化。
  - インサイトページの一部を第11回データで再確認し、第10回・第11回比較の特集ページを追加。
  - 外部AIアクセスを`/mcp`のRemote MCP、`/api/v1/*`のREST、`/mcp/openapi`のOpenAPIへ整理し、direct DB・旧bridge・任意SQL経路を退役。
- 2026-09（Tursoへの移行）
  - Supabaseの公開対象データをTursoへ移行し、Vercel上の公開サイトの参照先を切り替え。
  - データ比較・代表条件の確認と本番の軽量テストを完了。既存の公開URL・API・MCPを継続。

データ実装の到達点（抜粋）
- 質問票データ: 第10回 278,334レコード、第11回 278,816レコード。
- 検査データ: 第10回 467,663レコード相当、第11回 509,677レコード相当（基本/詳細の2層構造）。
- 第11回は検査76ファイル、質問票44ファイルを確認し、平均値4ファイルをDB投入対象外として扱っています。

### 🧪 品質の考え方
- Excel取込時は4点突合で、Excelセル値＝DB格納値＝API応答値＝UI表示値の同一性を確認。2026年9月のDB移行では、全件データ比較と代表条件の確認を受入基準に採用。
- 年齢層・項目名・地域名の正規化を徹底（例: 40-44／GOT/GPT/γ-GT／都道府県-圏域名）。

<a id="dev-method"></a>
## 🛠️ 開発手法（AI協働）

- 利用ツール: Cursor／Claude Code／Codex
- 実装分担: 設計・コーディング・テストの実装作業は原則 99.9% をAIが実行
- 意思決定: 企画・要件・アーキテクチャ上の意思決定は 100% 人間が実施

### 参考換算（従来工法の「ステップ数」と人月）
- 有効コード量（ライブラリ除外）
  - TypeScript/TSX（viewer-frontend/src）: 約 22,856 行
  - Python（scripts, tests 主要部）: 約 96,014 行
  - SQL（supabase/migrations, scripts/masters）: 約 3,628 行
  - Shell（scripts 配下）: 約 5,188 行
  - 合計: 約 127,700 ステップ（行）
- 人月換算（目安）
  - 従来工法の一般的生産性: 3,000〜6,000 行／人月
  - 推定規模: 約 21〜43 人月相当（中心値 ≈ 32 人月）

注:
- 上記は生成物の有効コードに限定し、外部ライブラリや生成キャッシュ、ビルド成果物は含みません。
- 実プロジェクトではAIが大半を実装しており、従来工法の人月は“比較指標”としての参考値です。

<a id="contact"></a>
## 📬 お問い合わせ
- 不具合報告・質問・ご提案は GitHub Issues へお願いします: https://github.com/rx-tomo/ndbopendata-hub/issues
- メール: toms_pjt-ndbopendata@yahoo.co.jp


## 📎 関連ドキュメント
- データ構造の解説: ./reference/DATA_STRUCTURE.md
- Excel個別読み取りシステム: ./reference/README_excel_readers.md
- 質問・課題の報告: GitHub Issues（本リポジトリ）

---

このリポジトリは公開用の抜粋ドキュメントのみを含みます。

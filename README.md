# ndbopendata-hub

第10回NDB（ナショナルデータベース）オープンデータのうち、特定健診の「質問票」「検査」関連を対象にした個人開発した公開ハブです。

本リポジトリに含まれるデータやサンプルは、いずれもNDBオープンデータで公開されている統計上の集計値であり、個人情報を含みません。

— Quick links —
- [APIエンドポイント](#api-endpoints)
- [開発の進行概要](#progress)
- [データ構造](#data-structure)
- [データ処理アーキテクチャ](#processing-arch)
- [データベース構成](#db-schema)
- [開発手法（AI協働）](#dev-method)
- [お問い合わせ](#contact)

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
curl -s "https://ndbopendata-hub.com/api/v1/range-labels?item_name=BMI&record_mode=basic"

# 2) 取得した range_id を使って人数を取得
curl -s "https://ndbopendata-hub.com/api/v1/inspection-stats?item_name=BMI&record_mode=basic&area_type=prefecture&prefecture_code=02&gender=M&age_group=40-44&value_range=142"
```

レスポンスにはレート制限ヘッダ（`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`）を付与しています。

## 📊 対象データセット

- 名称: 第10回NDBオープンデータ
- 公開年度: 令和5年度（2023年度）
- データ内容:
  - 令和4年度（2022年度）に実施された特定健診の情報
  - 特定健診質問票（22項目）
  - 特定健診検査データ（基本情報レコード25項目・詳細情報レコード11項目）
- データベース上の data_year: `2023`（データセット公開年度の識別子）

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

### 特定健診質問票系
- `questionnaire_questions`: 質問項目マスタ（22問）
- `questionnaire_answer_options`: 回答選択肢マスタ
- `questionnaire_responses`: 回答データ（都道府県別・二次医療圏別）
- `questionnaire_import_history`: インポート履歴

### 特定健診検査データ系
- `health_inspection_items`: 検査項目マスタ（27項目）
- `health_inspection_value_ranges`: 検査値階層マスタ（性別対応を含む）
- `basic_checkup_results`: 基本情報レコードの検査結果（地域・性別・年齢層・値範囲ごとの集計）
- `detailed_checkup_results`: 詳細情報レコードの検査結果（医師判定ありの層）
- `health_inspection_import_history`: インポート履歴

### 地域マスタ
- `prefectures`: 都道府県マスタ
- `secondary_medical_areas`: 二次医療圏マスタ

<a id="progress"></a>
## 🔄 開発の進行概要（2025年9月30日）

- 2025-06（初期）
  - 仮のWebページ開発（プロトタイプUI／試験的API）。NDB第10回データの基本設計・DB正規化・初期の参照用画面を短期で構築。
  - MCPもプロトタイプ（PostgreSQL MCP Server と LLM 接続の試行）。
- 2025-07（基盤強化）
  - Excel個別リーダー方式へ全面転換（1ファイル=1スクリプト）。TDD確立、4点突合（Excel/DB/API/UI）の品質ゲートを導入。
- 2025-09（公開用リニューアル）
  - Webを公開前提で再設計（UI整理、公開ドキュメント整備、配布戦略の明文化）。
  - MCPをHTTP/OpenAPIで外部公開可能な形に拡張（Actions/Connectorの導線・レート制限・CORS等）。

データ実装の到達点（抜粋）
- 質問票データ: 22問・277,640レコード（延べ12億回答超）。Excel↔DB 100%整合を達成。
- 検査データ: 27項目・74ファイル・157,693レコード（基本/詳細の2層構造）。

### 🧪 品質の考え方
- 4点突合で、Excelセル値＝DB格納値＝API応答値＝UI表示値の同一性を標準化。
- 年齢層・項目名・地域名の正規化を徹底（例: 40-44／GOT/GPT/γ-GT／都道府県-圏域名）。

<a id="dev-method"></a>
## 🛠️ 開発手法（AI協働）

- 利用ツール: Cursor／Claude Code／Codex
- 実装分担: 設計・コーディング・テストの実装作業は原則 99.9% をAIが実行
- 意思決定: 企画・要件・アーキテクチャ上の意思決定は 100% 人間が実施

### 参考換算（従来工法の「ステップ数」と人月）
- 有効コード量（ライブラリ除外）
  - TypeScript/TSX（viewer-frontend/src, mcp-server/src）: 約 22,856 行
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

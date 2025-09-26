# MCP ブリッジ（スタンドアロン版）配布・利用ガイド

目的: Node.js をインストールせずに、Claude Desktop などのMCPクライアントから本サービスのMCPサーバ（/api/mcp）へ接続できるよう、スタンドアロン実行ファイル（バイナリ）を配布します。

## 1. 配布物

- 配布形態: GitHub Actions のアーティファクト／GitHub Release アセット
- ファイル例（自動生成; OS/CPU別）
  - `ndb-mcp-bridge-macos-arm64`
  - `ndb-mcp-bridge-macos-x64`
  - `ndb-mcp-bridge-linux-x64`
  - `ndb-mcp-bridge-win-x64.exe`
  - `checksums.txt`（SHA256）

> 最新の配布物は、GitHub の Actions 実行結果から「mcp-bridge-dist」をダウンロード、またはタグを付けたリリースのアセットから取得してください。

## 2. 使い方（Claude Desktop）

1) 実行ファイルを任意のフォルダへ配置（例: `~/Tools/ndb-mcp-bridge`）
2) 権限付与（macOS/Linux）
```
chmod +x ndb-mcp-bridge-macos-arm64
```
3) Claude 設定（macOS） `~/Library/Application Support/Claude/claude_desktop_config.json` に登録
```json
{
  "mcpServers": {
    "ndb-mcp-bridge": {
      "command": "/Users/you/Tools/ndb-mcp-bridge/ndb-mcp-bridge-macos-arm64",
      "args": [],
      "env": {
        "NDB_MCP_SERVER_URL": "https://ndbopendata-hub.com/api/mcp",
        "MCP_DEBUG": "false"
      }
    }
  }
}
```
4) Claude Desktop を完全再起動 → ツール一覧で NDB 関連ツールが表示されることを確認

## 3. 動作原理（概要）
- ブリッジは stdio トランスポートでMCPリクエストを受け取り、HTTPの `/api/mcp` へ JSON‑RPC を転送します。
- DB資格情報は保持せず、公開HTTP先へ読み取り専用でプロキシします。

## 4. 検証と整合性
- 署名: `checksums.txt` の SHA256 を検証してください。
- ネットワーク要件: `https://ndbopendata-hub.com/api/mcp` へ到達できること。

## 5. よくある質問（FAQ）
- Q: Node.js は必要ですか？
  - A: 本バイナリ版は不要です。ダウンロードしてそのまま実行できます。
- Q: Windowsでの設定は？
  - A: `command` に `.../ndb-mcp-bridge-win-x64.exe` を指定し、`args` は空でOKです。
- Q: URL直指定で使えませんか？
  - A: Claude はローカル実行（stdio）を起動する設計のため、本ブリッジかローカルMCPサーバが必要です。ChatGPT Actions を使う場合は `/mcp/openapi` のURL登録のみで動作します。

## 6. 管理者向け: 配布の作り方
- ローカル: `cd mcp-bridge && npm ci && npm run release:bin`（`dist/` に生成、`checksums.txt` 付き）
- CI（自動）: `.github/workflows/build-bridge-binaries.yml` が push/dispatch で実行し、Artifacts として配布。タグを打つと Release に添付されます。

## 7. 既知の制限
- 企業内プロキシ／証明書検証エラー時は、OSの証明書ストア設定に依存します。
- 本ブリッジは読み取り専用の前提です。将来の書込み系ツールは別途検討します。


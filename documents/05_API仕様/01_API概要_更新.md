# API概要: LLM学術文書支援システム

## 1. API基本情報

- **ベースURL**: `https://api.example.com/v1`
- **開発環境URL**: `http://localhost:8000/v1`
- **形式**: RESTful API + WebSocket（リアルタイム通信用）
- **認証方式**: JWT (JSON Web Token)
- **レスポンス形式**: JSON
- **リクエスト形式**: JSON または multipart/form-data（ファイルアップロード時）

## 2. API構成

本システムのAPIは以下の主要カテゴリに分類されます：

| カテゴリ | 説明 | 主要エンドポイント |
|----------|------|-------------------|
| 認証 | ユーザー登録・ログイン | `/auth/register`, `/auth/login` |
| プロジェクト | プロジェクト管理 | `/projects`, `/projects/{id}` |
| 文書 | 文書管理とアップロード | `/projects/{id}/documents` |
| セクション | 文書の階層構造 | `/documents/{id}/sections` |
| LLM | LLMとの対話 | `/llm/chat`, `/llm/stream` |
| ノート | 学習ノート生成・管理 | `/notes` |
| 検索 | コンテンツ検索 | `/search` |

## 3. OpenAPI仕様

システムのAPIは、OpenAPI (Swagger) 仕様で定義されています。詳細は以下のファイルを参照してください：

- **統合API仕様**: [03_OpenAPI仕様_統合.yaml](./03_OpenAPI仕様_統合.yaml)

この仕様書は、APIのすべてのエンドポイント、リクエスト・レスポンス形式、認証方法などを包括的に定義しています。従来の複数ファイルから統合されました。

## 4. 認証フロー

APIでは、以下の認証フローを採用しています：

1. ユーザーが `/auth/login` エンドポイントにログイン情報を送信
2. 認証成功時、サーバーがJWTトークンを発行
3. 以降のAPI呼び出しで、ヘッダーに `Authorization: Bearer {token}` を付与
4. トークンは有効期限切れまたはログアウトするまで有効

## 5. エラーハンドリング

APIは、標準的なHTTPステータスコードと詳細なエラーレスポンスを返します：

```json
{
  "code": "INVALID_INPUT",
  "message": "プロジェクト名は必須です",
  "details": {
    "field": "name",
    "constraint": "required"
  }
}
```

主なエラーコード：

- `400 Bad Request`: 不正なリクエスト形式またはパラメータ
- `401 Unauthorized`: 認証エラー
- `403 Forbidden`: 認可エラー（権限不足）
- `404 Not Found`: リソースが存在しない
- `409 Conflict`: 競合状態（既に存在するリソースの重複作成など）
- `500 Internal Server Error`: サーバー内部エラー

## 6. APIバージョニング

- URLパスの最初の部分でバージョンを指定（例: `/v1/projects`）
- 主要な破壊的変更の際に、新しいバージョンを導入
- 後方互換性を保つための移行期間を設定

## 7. レート制限

システムではAPI呼び出しに制限を設けています：

- 認証済みユーザー: 100リクエスト/分
- 未認証ユーザー: 20リクエスト/分
- LLM関連エンドポイント: 30リクエスト/分

制限超過時は `429 Too Many Requests` レスポンスを返します。

## 8. 関連資料

- [エンドポイント一覧](./02_エンドポイント一覧.md)
- [APIリファレンス詳細](./04_APIリファレンス詳細)

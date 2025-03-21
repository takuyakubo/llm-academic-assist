# API概要

## はじめに

LLM学術文書支援システムのAPIは、フロントエンドアプリケーションとバックエンドサービス間の通信を行うための標準化されたインターフェースを提供します。本ドキュメントでは、システムが提供するAPIの全体像と設計思想について説明します。

## API設計方針

### RESTful設計

本システムのAPIは、RESTful原則に従って設計されています。主な方針は以下の通りです：

- リソース指向の設計
- HTTPメソッドの適切な使用（GET, POST, PUT, DELETE）
- ステートレスな通信
- 標準的なHTTPステータスコードの使用
- JSON形式でのデータ交換

### エンドポイント命名規則

```
/api/{リソース}/{ID}/{サブリソース}
```

例：
- `/api/projects` - プロジェクト一覧の取得
- `/api/projects/123` - 特定のプロジェクトの取得
- `/api/projects/123/documents` - プロジェクト内のドキュメント一覧の取得

### バージョニング

APIの後方互換性を維持しながら進化させるため、URLベースのバージョニングを採用しています。

```
/api/v1/projects
```

ただし、初期リリースでは省略して `/api/projects` の形式を使用します。

## 認証・認可

### 認証方式

JWTベースの認証を採用しています。認証フローは以下の通りです：

1. ユーザーがログイン情報を送信
2. サーバーがJWTトークンを発行
3. クライアントは以降のリクエストで `Authorization` ヘッダーにトークンを含める

```
Authorization: Bearer {JWTトークン}
```

### 認可

リソースへのアクセス権限はロールベースで制御されます。主なロールは以下の通りです：

- **管理者（Admin）**: すべての操作が可能
- **プロジェクト所有者（Owner）**: 特定プロジェクトの完全な制御権限
- **編集者（Editor）**: ドキュメントの追加・編集が可能
- **閲覧者（Viewer）**: ドキュメントの閲覧のみ可能

## APIカテゴリ

本システムのAPIは以下のカテゴリに分類されます：

1. **認証API**: ユーザー認証と認可を管理
2. **プロジェクトAPI**: プロジェクトの作成・管理
3. **ドキュメントAPI**: 文書のアップロード・処理・取得
4. **LLM API**: LLMを使用した質問応答や要約生成
5. **ノートAPI**: 学習ノートの作成・管理
6. **検索API**: 文書やプロジェクトの検索

## レスポンス形式

すべてのAPIレスポンスは一貫した形式で返されます：

```json
{
  "success": true,
  "data": {
    // レスポンスデータ
  },
  "meta": {
    // ページネーション情報など
  }
}
```

エラーの場合：

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "エラーメッセージ",
    "details": {
      // 詳細なエラー情報
    }
  }
}
```

## エラーハンドリング

一般的なHTTPステータスコードに加えて、アプリケーション固有のエラーコードを使用します。

### 主なHTTPステータスコード

- **200 OK**: リクエスト成功
- **201 Created**: リソース作成成功
- **400 Bad Request**: リクエスト形式エラー
- **401 Unauthorized**: 認証エラー
- **403 Forbidden**: 権限エラー
- **404 Not Found**: リソースが存在しない
- **500 Internal Server Error**: サーバー内部エラー

### アプリケーションエラーコード

- **AUTH_001**: 無効な認証情報
- **AUTH_002**: アクセストークン期限切れ
- **PROJ_001**: プロジェクト作成エラー
- **DOC_001**: ドキュメント処理エラー
- **LLM_001**: LLM API呼び出しエラー

## レート制限

APIの安定性と公平な利用を確保するため、レート制限を実装しています：

- 認証済みユーザー: 60リクエスト/分
- LLM関連エンドポイント: 20リクエスト/分

レート制限情報はレスポンスヘッダーに含まれます：

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1632924800
```

## API仕様ドキュメント

詳細なAPI仕様は、OpenAPI（Swagger）形式で提供されており、以下の方法でアクセスできます：

- 開発環境: `http://localhost:3000/api-docs`
- 本番環境: `https://example.com/api-docs`

## サンプルリクエスト

### プロジェクト作成

```bash
curl -X POST https://example.com/api/projects \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "量子力学研究",
    "description": "量子力学の基本原理と応用に関する研究プロジェクト"
  }'
```

### ドキュメントアップロード

```bash
curl -X POST https://example.com/api/projects/123/documents \
  -H "Authorization: Bearer {token}" \
  -F "file=@quantum_paper.pdf" \
  -F "title=量子コンピューティングの基礎" \
  -F "description=量子コンピューティングの基本概念と最新の研究動向"
```

### LLM質問応答

```bash
curl -X POST https://example.com/api/llm/chat \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "documentId": "456",
    "message": "シュレディンガー方程式の物理的な意味を説明してください"
  }'
```

# プロジェクトAPI

このドキュメントでは、LLM学術文書支援システムのプロジェクト管理APIについて詳細に説明します。

## エンドポイント一覧

| メソッド | エンドポイント | 説明 |
|---------|---------------|------|
| GET | `/api/projects` | プロジェクト一覧取得 |
| POST | `/api/projects` | プロジェクト作成 |
| GET | `/api/projects/{projectId}` | プロジェクト詳細取得 |
| PUT | `/api/projects/{projectId}` | プロジェクト更新 |
| DELETE | `/api/projects/{projectId}` | プロジェクト削除 |
| POST | `/api/projects/{projectId}/collaborators` | 共同編集者追加 |
| DELETE | `/api/projects/{projectId}/collaborators/{userId}` | 共同編集者削除 |
| PUT | `/api/projects/{projectId}/collaborators/{userId}` | 共同編集者ロール更新 |
| GET | `/api/projects/{projectId}/activity` | プロジェクトアクティビティ取得 |

## プロジェクト一覧取得

**エンドポイント**: `GET /api/projects`

ユーザーのプロジェクト一覧を取得します。デフォルトでは、ユーザーが所有者または共同編集者であるプロジェクトのみが返されます。

### リクエストパラメータ

| パラメータ | 型 | 説明 | デフォルト |
|------------|------|--------------|----------|
| page | integer | ページ番号（1から開始） | 1 |
| limit | integer | 1ページあたりの件数 | 10 |
| sort | string | ソート項目（name, createdAt, updatedAt） | createdAt |
| order | string | ソート順序（asc, desc） | desc |
| query | string | 検索クエリ（プロジェクト名、説明、タグを検索） | null |
| tags | string | カンマ区切りのタグ一覧でフィルタリング | null |
| includeShared | boolean | 共有プロジェクトを含めるかどうか | true |

### レスポンス

```json
{
  "success": true,
  "data": [
    {
      "id": "507f1f77bcf86cd799439012",
      "name": "量子力学入門",
      "description": "量子力学の基礎概念についての学習プロジェクト",
      "ownerId": "507f1f77bcf86cd799439011",
      "collaborators": [
        {
          "userId": "507f1f77bcf86cd799439013",
          "role": "editor",
          "addedAt": "2023-02-10T14:20:00.000Z"
        }
      ],
      "tags": ["物理学", "量子力学", "入門"],
      "createdAt": "2023-02-01T09:15:00.000Z",
      "updatedAt": "2023-03-15T11:30:00.000Z",
      "progress": {
        "documentCount": 3,
        "completedSections": 12,
        "totalSections": 30
      }
    },
    // 追加のプロジェクト...
  ],
  "meta": {
    "currentPage": 1,
    "totalPages": 3,
    "totalItems": 25,
    "itemsPerPage": 10
  }
}
```

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_003 | 認証エラー（無効なトークン） |
| PARAM_001 | 無効なクエリパラメータ |
| SERVER_001 | サーバー内部エラー |

## プロジェクト作成

**エンドポイント**: `POST /api/projects`

新しいプロジェクトを作成します。

### リクエスト

```json
{
  "name": "相対性理論研究",
  "description": "アインシュタインの特殊相対性理論と一般相対性理論の学習プロジェクト",
  "tags": ["物理学", "相対性理論", "アインシュタイン"],
  "settings": {
    "defaultLLMModel": "claude-3-opus",
    "publiclyViewable": false,
    "customPromptTemplates": [
      {
        "name": "概念説明",
        "template": "相対性理論における{{concept}}の概念を、高校生が理解できるように簡潔に説明してください。"
      }
    ]
  }
}
```

### リクエスト本文パラメータ

| パラメータ | 型 | 説明 | 必須 |
|------------|------|--------------|--------|
| name | string | プロジェクト名（最大100文字） | はい |
| description | string | プロジェクトの説明（最大1000文字） | いいえ |
| tags | array<string> | プロジェクトに関連するタグ | いいえ |
| settings | object | プロジェクト設定 | いいえ |
| settings.defaultLLMModel | string | デフォルトのLLMモデル | いいえ |
| settings.publiclyViewable | boolean | 公開プロジェクトかどうか | いいえ |
| settings.customPromptTemplates | array<object> | カスタムプロンプトテンプレート | いいえ |

### レスポンス

```json
{
  "success": true,
  "data": {
    "id": "507f1f77bcf86cd799439030",
    "name": "相対性理論研究",
    "description": "アインシュタインの特殊相対性理論と一般相対性理論の学習プロジェクト",
    "ownerId": "507f1f77bcf86cd799439011",
    "collaborators": [],
    "tags": ["物理学", "相対性理論", "アインシュタイン"],
    "settings": {
      "defaultLLMModel": "claude-3-opus",
      "publiclyViewable": false,
      "customPromptTemplates": [
        {
          "id": "507f1f77bcf86cd799439031",
          "name": "概念説明",
          "template": "相対性理論における{{concept}}の概念を、高校生が理解できるように簡潔に説明してください。"
        }
      ]
    },
    "createdAt": "2023-04-22T10:15:00.000Z",
    "updatedAt": "2023-04-22T10:15:00.000Z",
    "progress": {
      "documentCount": 0,
      "completedSections": 0,
      "totalSections": 0
    }
  }
}
```

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_003 | 認証エラー（無効なトークン） |
| VALIDATION_001 | 入力バリデーションエラー |
| QUOTA_001 | プロジェクト数の上限に達しました |
| SERVER_001 | サーバー内部エラー |

## プロジェクト詳細取得

**エンドポイント**: `GET /api/projects/{projectId}`

特定のプロジェクトの詳細情報を取得します。

### パスパラメータ

| パラメータ | 型 | 説明 | 必須 |
|------------|------|--------------|--------|
| projectId | string | プロジェクトID | はい |

### レスポンス

```json
{
  "success": true,
  "data": {
    "id": "507f1f77bcf86cd799439012",
    "name": "量子力学入門",
    "description": "量子力学の基礎概念についての学習プロジェクト",
    "ownerId": "507f1f77bcf86cd799439011",
    "collaborators": [
      {
        "userId": "507f1f77bcf86cd799439013",
        "displayName": "田中花子",
        "role": "editor",
        "addedAt": "2023-02-10T14:20:00.000Z"
      }
    ],
    "settings": {
      "defaultLLMModel": "claude-3-opus",
      "customPromptTemplates": [
        {
          "id": "507f1f77bcf86cd799439031",
          "name": "数式解説",
          "template": "以下の数式について、高校生にもわかるように噛み砕いて説明してください: {{equation}}"
        }
      ],
      "publiclyViewable": false
    },
    "tags": ["物理学", "量子力学", "入門"],
    "createdAt": "2023-02-01T09:15:00.000Z",
    "updatedAt": "2023-03-15T11:30:00.000Z",
    "progress": {
      "documentCount": 3,
      "completedSections": 12,
      "totalSections": 30
    },
    "recentActivity": [
      {
        "type": "document_added",
        "documentId": "507f1f77bcf86cd799439014",
        "title": "シュレディンガー方程式の導出と応用",
        "timestamp": "2023-03-15T10:20:00.000Z"
      },
      {
        "type": "note_created",
        "noteId": "507f1f77bcf86cd799439019",
        "title": "シュレディンガー方程式の要点まとめ",
        "timestamp": "2023-03-15T09:40:00.000Z"
      }
    ]
  }
}
```

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_003 | 認証エラー（無効なトークン） |
| AUTH_010 | アクセス権限がありません |
| RESOURCE_001 | プロジェクトが見つかりません |
| SERVER_001 | サーバー内部エラー |

## プロジェクト更新

**エンドポイント**: `PUT /api/projects/{projectId}`

特定のプロジェクトの情報を更新します。プロジェクトの所有者または編集権限を持つ共同編集者のみが実行できます。

### パスパラメータ

| パラメータ | 型 | 説明 | 必須 |
|------------|------|--------------|--------|
| projectId | string | プロジェクトID | はい |

### リクエスト本文

```json
{
  "name": "量子力学基礎と応用",
  "description": "量子力学の基本概念から実際の応用例までを網羅的に学ぶプロジェクト",
  "tags": ["物理学", "量子力学", "基礎", "応用"],
  "settings": {
    "defaultLLMModel": "claude-3-opus",
    "publiclyViewable": true,
    "customPromptTemplates": [
      {
        "id": "507f1f77bcf86cd799439031",
        "name": "数式解説（詳細）",
        "template": "以下の数式について、高校生にもわかるように詳細に解説してください。数式の物理的な意味、導出過程、および具体的な応用例も含めてください: {{equation}}"
      },
      {
        "name": "簡易解説",
        "template": "以下の概念について簡潔に説明してください: {{concept}}"
      }
    ]
  }
}
```

### リクエスト本文パラメータ

| パラメータ | 型 | 説明 | 必須 |
|------------|------|--------------|--------|
| name | string | プロジェクト名（最大100文字） | いいえ |
| description | string | プロジェクトの説明（最大1000文字） | いいえ |
| tags | array<string> | プロジェクトに関連するタグ | いいえ |
| settings | object | プロジェクト設定 | いいえ |

### レスポンス

```json
{
  "success": true,
  "data": {
    "id": "507f1f77bcf86cd799439012",
    "name": "量子力学基礎と応用",
    "description": "量子力学の基本概念から実際の応用例までを網羅的に学ぶプロジェクト",
    "ownerId": "507f1f77bcf86cd799439011",
    "collaborators": [
      {
        "userId": "507f1f77bcf86cd799439013",
        "role": "editor",
        "addedAt": "2023-02-10T14:20:00.000Z"
      }
    ],
    "tags": ["物理学", "量子力学", "基礎", "応用"],
    "settings": {
      "defaultLLMModel": "claude-3-opus",
      "publiclyViewable": true,
      "customPromptTemplates": [
        {
          "id": "507f1f77bcf86cd799439031",
          "name": "数式解説（詳細）",
          "template": "以下の数式について、高校生にもわかるように詳細に解説してください。数式の物理的な意味、導出過程、および具体的な応用例も含めてください: {{equation}}"
        },
        {
          "id": "507f1f77bcf86cd799439032",
          "name": "簡易解説",
          "template": "以下の概念について簡潔に説明してください: {{concept}}"
        }
      ]
    },
    "createdAt": "2023-02-01T09:15:00.000Z",
    "updatedAt": "2023-04-22T11:45:00.000Z",
    "progress": {
      "documentCount": 3,
      "completedSections": 12,
      "totalSections": 30
    }
  }
}
```

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_003 | 認証エラー（無効なトークン） |
| AUTH_010 | アクセス権限がありません |
| RESOURCE_001 | プロジェクトが見つかりません |
| VALIDATION_001 | 入力バリデーションエラー |
| SERVER_001 | サーバー内部エラー |

## プロジェクト削除

**エンドポイント**: `DELETE /api/projects/{projectId}`

特定のプロジェクトを削除します。プロジェクトの所有者のみが実行できます。

### パスパラメータ

| パラメータ | 型 | 説明 | 必須 |
|------------|------|--------------|--------|
| projectId | string | プロジェクトID | はい |

### レスポンス

```json
{
  "success": true,
  "data": {
    "message": "プロジェクトが正常に削除されました"
  }
}
```

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_003 | 認証エラー（無効なトークン） |
| AUTH_010 | アクセス権限がありません |
| RESOURCE_001 | プロジェクトが見つかりません |
| SERVER_001 | サーバー内部エラー |

## 共同編集者追加

**エンドポイント**: `POST /api/projects/{projectId}/collaborators`

プロジェクトに共同編集者を追加します。プロジェクトの所有者のみが実行できます。

### パスパラメータ

| パラメータ | 型 | 説明 | 必須 |
|------------|------|--------------|--------|
| projectId | string | プロジェクトID | はい |

### リクエスト本文

```json
{
  "email": "collaborator@example.com",
  "role": "editor"
}
```

### リクエスト本文パラメータ

| パラメータ | 型 | 説明 | 必須 |
|------------|------|--------------|--------|
| email | string | 共同編集者のメールアドレス | はい |
| role | string | ロール（viewer, editor） | はい |

### レスポンス

```json
{
  "success": true,
  "data": {
    "userId": "507f1f77bcf86cd799439055",
    "displayName": "佐藤三郎",
    "email": "collaborator@example.com",
    "role": "editor",
    "addedAt": "2023-04-22T15:30:00.000Z"
  }
}
```

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_003 | 認証エラー（無効なトークン） |
| AUTH_010 | アクセス権限がありません |
| RESOURCE_001 | プロジェクトが見つかりません |
| RESOURCE_002 | ユーザーが見つかりません |
| VALIDATION_001 | 入力バリデーションエラー |
| COLLAB_001 | ユーザーは既にプロジェクトの共同編集者です |
| SERVER_001 | サーバー内部エラー |

## 共同編集者削除

**エンドポイント**: `DELETE /api/projects/{projectId}/collaborators/{userId}`

プロジェクトから共同編集者を削除します。プロジェクトの所有者のみが実行できます。

### パスパラメータ

| パラメータ | 型 | 説明 | 必須 |
|------------|------|--------------|--------|
| projectId | string | プロジェクトID | はい |
| userId | string | 共同編集者のユーザーID | はい |

### レスポンス

```json
{
  "success": true,
  "data": {
    "message": "共同編集者が正常に削除されました"
  }
}
```

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_003 | 認証エラー（無効なトークン） |
| AUTH_010 | アクセス権限がありません |
| RESOURCE_001 | プロジェクトが見つかりません |
| RESOURCE_002 | ユーザーが見つかりません |
| COLLAB_002 | ユーザーはプロジェクトの共同編集者ではありません |
| SERVER_001 | サーバー内部エラー |

## 共同編集者ロール更新

**エンドポイント**: `PUT /api/projects/{projectId}/collaborators/{userId}`

プロジェクトの共同編集者のロールを更新します。プロジェクトの所有者のみが実行できます。

### パスパラメータ

| パラメータ | 型 | 説明 | 必須 |
|------------|------|--------------|--------|
| projectId | string | プロジェクトID | はい |
| userId | string | 共同編集者のユーザーID | はい |

### リクエスト本文

```json
{
  "role": "viewer"
}
```

### リクエスト本文パラメータ

| パラメータ | 型 | 説明 | 必須 |
|------------|------|--------------|--------|
| role | string | 新しいロール（viewer, editor） | はい |

### レスポンス

```json
{
  "success": true,
  "data": {
    "userId": "507f1f77bcf86cd799439055",
    "displayName": "佐藤三郎",
    "email": "collaborator@example.com",
    "role": "viewer",
    "addedAt": "2023-04-22T15:30:00.000Z",
    "updatedAt": "2023-04-22T16:45:00.000Z"
  }
}
```

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_003 | 認証エラー（無効なトークン） |
| AUTH_010 | アクセス権限がありません |
| RESOURCE_001 | プロジェクトが見つかりません |
| RESOURCE_002 | ユーザーが見つかりません |
| COLLAB_002 | ユーザーはプロジェクトの共同編集者ではありません |
| VALIDATION_001 | 入力バリデーションエラー |
| SERVER_001 | サーバー内部エラー |

## プロジェクトアクティビティ取得

**エンドポイント**: `GET /api/projects/{projectId}/activity`

プロジェクトの最近のアクティビティを取得します。

### パスパラメータ

| パラメータ | 型 | 説明 | 必須 |
|------------|------|--------------|--------|
| projectId | string | プロジェクトID | はい |

### リクエストパラメータ

| パラメータ | 型 | 説明 | デフォルト |
|------------|------|--------------|----------|
| page | integer | ページ番号（1から開始） | 1 |
| limit | integer | 1ページあたりの件数 | 20 |
| type | string | アクティビティタイプでフィルタリング | null |
| startDate | string | 開始日（ISO形式） | null |
| endDate | string | 終了日（ISO形式） | null |

### レスポンス

```json
{
  "success": true,
  "data": [
    {
      "id": "507f1f77bcf86cd799439060",
      "type": "document_added",
      "documentId": "507f1f77bcf86cd799439014",
      "title": "シュレディンガー方程式の導出と応用",
      "userId": "507f1f77bcf86cd799439011",
      "userDisplayName": "山田太郎",
      "timestamp": "2023-03-15T10:20:00.000Z"
    },
    {
      "id": "507f1f77bcf86cd799439061",
      "type": "note_created",
      "noteId": "507f1f77bcf86cd799439019",
      "title": "シュレディンガー方程式の要点まとめ",
      "userId": "507f1f77bcf86cd799439011",
      "userDisplayName": "山田太郎",
      "timestamp": "2023-03-15T09:40:00.000Z"
    },
    {
      "id": "507f1f77bcf86cd799439062",
      "type": "collaborator_added",
      "collaboratorId": "507f1f77bcf86cd799439013",
      "collaboratorDisplayName": "田中花子",
      "role": "editor",
      "userId": "507f1f77bcf86cd799439011",
      "userDisplayName": "山田太郎",
      "timestamp": "2023-02-10T14:20:00.000Z"
    },
    // 追加のアクティビティ...
  ],
  "meta": {
    "currentPage": 1,
    "totalPages": 2,
    "totalItems": 32,
    "itemsPerPage": 20
  }
}
```

### アクティビティタイプ

| タイプ | 説明 |
|--------|------|
| project_created | プロジェクト作成 |
| project_updated | プロジェクト更新 |
| document_added | ドキュメント追加 |
| document_updated | ドキュメント更新 |
| document_deleted | ドキュメント削除 |
| note_created | ノート作成 |
| note_updated | ノート更新 |
| note_deleted | ノート削除 |
| collaborator_added | 共同編集者追加 |
| collaborator_updated | 共同編集者ロール更新 |
| collaborator_removed | 共同編集者削除 |
| conversation_created | 会話開始 |

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_003 | 認証エラー（無効なトークン） |
| AUTH_010 | アクセス権限がありません |
| RESOURCE_001 | プロジェクトが見つかりません |
| PARAM_001 | 無効なクエリパラメータ |
| SERVER_001 | サーバー内部エラー |
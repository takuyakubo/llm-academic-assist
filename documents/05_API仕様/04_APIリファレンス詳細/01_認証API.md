# 認証API

このドキュメントでは、LLM学術文書支援システムの認証APIについて詳細に説明します。

## エンドポイント一覧

| メソッド | エンドポイント | 説明 |
|---------|---------------|------|
| POST | `/api/auth/login` | ユーザーログイン |
| POST | `/api/auth/register` | ユーザー登録 |
| GET | `/api/auth/validate` | トークン検証 |
| POST | `/api/auth/refresh` | トークンリフレッシュ |
| POST | `/api/auth/logout` | ログアウト |
| POST | `/api/auth/password/reset/request` | パスワードリセットリクエスト |
| POST | `/api/auth/password/reset` | パスワードリセット実行 |

## ユーザーログイン

**エンドポイント**: `POST /api/auth/login`

ユーザーのメールアドレスとパスワードを使用して認証を行い、JWTトークンを取得します。

### リクエスト

```json
{
  "email": "user@example.com",
  "password": "securePassword123"
}
```

### レスポンス (成功)

```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "507f1f77bcf86cd799439011",
      "email": "user@example.com",
      "displayName": "山田太郎",
      "role": "user",
      "createdAt": "2023-01-15T08:30:00.000Z",
      "settings": {
        "theme": "light",
        "language": "ja",
        "notifications": true
      }
    }
  }
}
```

### レスポンス (失敗)

```json
{
  "success": false,
  "error": {
    "code": "AUTH_001",
    "message": "メールアドレスまたはパスワードが正しくありません"
  }
}
```

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_001 | 認証失敗（無効なメールアドレスまたはパスワード） |
| AUTH_010 | アカウントがロックされています |
| SERVER_001 | サーバー内部エラー |

## ユーザー登録

**エンドポイント**: `POST /api/auth/register`

新しいユーザーアカウントを作成します。

### リクエスト

```json
{
  "email": "newuser@example.com",
  "password": "securePassword123",
  "displayName": "鈴木一郎"
}
```

### レスポンス (成功)

```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "507f1f77bcf86cd799439025",
      "email": "newuser@example.com",
      "displayName": "鈴木一郎",
      "role": "user",
      "createdAt": "2023-04-20T14:30:00.000Z",
      "settings": {
        "theme": "light",
        "language": "ja",
        "notifications": true
      }
    }
  }
}
```

### レスポンス (失敗)

```json
{
  "success": false,
  "error": {
    "code": "AUTH_002",
    "message": "このメールアドレスは既に登録されています",
    "details": {
      "field": "email"
    }
  }
}
```

### バリデーションルール

| フィールド | ルール |
|-----------|--------|
| email | 有効なメールアドレス形式、必須 |
| password | 最小8文字、1つ以上の数字と特殊文字、必須 |
| displayName | 最小2文字、最大50文字、必須 |

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_002 | メールアドレスが既に使用されています |
| VALIDATION_001 | 入力バリデーションエラー |
| SERVER_001 | サーバー内部エラー |

## トークンの検証

**エンドポイント**: `GET /api/auth/validate`

現在のJWTトークンが有効かどうかを検証します。認証ヘッダーにトークンを含める必要があります。

### リクエスト

```
GET /api/auth/validate
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### レスポンス (成功)

```json
{
  "success": true,
  "data": {
    "valid": true,
    "user": {
      "id": "507f1f77bcf86cd799439011",
      "email": "user@example.com",
      "displayName": "山田太郎",
      "role": "user"
    }
  }
}
```

### レスポンス (失敗)

```json
{
  "success": false,
  "error": {
    "code": "AUTH_003",
    "message": "トークンが無効または期限切れです"
  }
}
```

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_003 | トークンが無効または期限切れ |
| AUTH_004 | トークンが見つかりません |
| SERVER_001 | サーバー内部エラー |

## トークンリフレッシュ

**エンドポイント**: `POST /api/auth/refresh`

リフレッシュトークンを使用して新しいアクセストークンを発行します。

### リクエスト

```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### レスポンス (成功)

```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

### レスポンス (失敗)

```json
{
  "success": false,
  "error": {
    "code": "AUTH_005",
    "message": "リフレッシュトークンが無効または期限切れです"
  }
}
```

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_005 | リフレッシュトークンが無効または期限切れ |
| AUTH_006 | リフレッシュトークンが見つかりません |
| SERVER_001 | サーバー内部エラー |

## ログアウト

**エンドポイント**: `POST /api/auth/logout`

ユーザーのリフレッシュトークンを無効化します。認証ヘッダーにアクセストークンを含める必要があります。

### リクエスト

```
POST /api/auth/logout
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### レスポンス (成功)

```json
{
  "success": true,
  "data": {
    "message": "正常にログアウトしました"
  }
}
```

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_003 | トークンが無効または期限切れ |
| AUTH_004 | トークンが見つかりません |
| SERVER_001 | サーバー内部エラー |

## パスワードリセットリクエスト

**エンドポイント**: `POST /api/auth/password/reset/request`

パスワードリセットメールの送信をリクエストします。

### リクエスト

```json
{
  "email": "user@example.com"
}
```

### レスポンス (成功)

```json
{
  "success": true,
  "data": {
    "message": "パスワードリセットメールを送信しました"
  }
}
```

> **注**: セキュリティ上の理由から、メールアドレスが登録されていない場合でも同じ成功レスポンスを返します。

### エラーコード

| コード | 説明 |
|--------|------|
| VALIDATION_001 | 入力バリデーションエラー |
| SERVER_001 | サーバー内部エラー |

## パスワードリセット実行

**エンドポイント**: `POST /api/auth/password/reset`

リセットトークンを使用してパスワードをリセットします。

### リクエスト

```json
{
  "token": "8f7e6d5c4b3a2910",
  "password": "newSecurePassword456"
}
```

### レスポンス (成功)

```json
{
  "success": true,
  "data": {
    "message": "パスワードが正常にリセットされました"
  }
}
```

### レスポンス (失敗)

```json
{
  "success": false,
  "error": {
    "code": "AUTH_007",
    "message": "リセットトークンが無効または期限切れです"
  }
}
```

### エラーコード

| コード | 説明 |
|--------|------|
| AUTH_007 | リセットトークンが無効または期限切れ |
| VALIDATION_001 | 入力バリデーションエラー (パスワードの形式が不正) |
| SERVER_001 | サーバー内部エラー |
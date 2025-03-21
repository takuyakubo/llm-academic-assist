# App Router レイアウト設計

## 概要

Next.js 14のApp Routerを使用したレイアウト設計について説明します。App Routerでは、ファイルシステムベースのルーティングに加えて、入れ子のレイアウトパターンを採用しています。

## レイアウト階層

```
app/
├── layout.tsx         # ルートレイアウト（全ページ共通）
├── (auth)/            # 認証関連のルートグループ
│   ├── layout.tsx     # 認証ページ共通レイアウト
│   ├── login/         # ログインページ
│   └── register/      # 登録ページ
├── (dashboard)/       # ダッシュボード関連のルートグループ
│   ├── layout.tsx     # ダッシュボード共通レイアウト
│   ├── page.tsx       # メインダッシュボード
│   ├── projects/      # プロジェクト一覧・管理
│   │   ├── page.tsx   # プロジェクト一覧
│   │   └── [id]/      # 動的ルート - 個別プロジェクト
│   │       ├── page.tsx            # プロジェクト詳細
│   │       ├── documents/          # プロジェクト内文書
│   │       │   ├── page.tsx        # 文書一覧
│   │       │   └── [docId]/        # 動的ルート - 個別文書
│   │       │       ├── page.tsx    # 文書詳細・閲覧
│   │       │       └── chat/       # 文書に関するチャット
│   │       │           └── page.tsx
│   │       └── notes/              # 学習ノート
│   │           └── page.tsx        # ノート一覧・編集
│   └── settings/      # ユーザー設定
│       └── page.tsx
└── api/               # API ルート
    └── [...]/         # 各種APIエンドポイント
```

## レイアウトコンポーネント設計

### ルートレイアウト

```tsx
// app/layout.tsx
export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="ja">
      <body>
        <ThemeProvider>
          <AuthProvider>
            {children}
          </AuthProvider>
        </ThemeProvider>
      </body>
    </html>
  );
}
```

### ダッシュボードレイアウト

```tsx
// app/(dashboard)/layout.tsx
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <div className="dashboard-container">
      <Sidebar />
      <main className="dashboard-content">
        <TopNavigation />
        <div className="content-area">
          {children}
        </div>
      </main>
    </div>
  );
}
```

## 並行ルートの活用

文書閲覧とチャットインターフェースを同時に表示する場合は、並行ルート（Parallel Routes）を活用します：

```tsx
// app/(dashboard)/projects/[id]/documents/[docId]/layout.tsx
export default function DocumentLayout({
  children,
  chat,
}: {
  children: React.ReactNode
  chat: React.ReactNode
}) {
  return (
    <div className="document-container">
      <div className="document-view">
        {children}
      </div>
      <div className="document-chat">
        {chat}
      </div>
    </div>
  );
}
```

```
app/(dashboard)/projects/[id]/documents/[docId]/
├── page.tsx            # 文書ビューア (@default)
└── @chat/              # 並行ルートスロット
    └── page.tsx        # チャットインターフェース
```

## クライアントとサーバーコンポーネントの使い分け

- サーバーコンポーネント（デフォルト）
  - データフェッチが必要なコンポーネント
  - SEOが重要なページ
  - 初期ロード速度を最適化したいコンポーネント

- クライアントコンポーネント（'use client'ディレクティブ）
  - インタラクティブなUI要素
  - ユーザーイベントのハンドリングが必要なもの
  - ブラウザAPIを使用するコンポーネント
  - 状態管理が必要なUI

## 効率的なロード状態の管理

```tsx
// app/(dashboard)/projects/[id]/loading.tsx
export default function Loading() {
  return (
    <div className="loading-skeleton">
      <div className="project-header-skeleton" />
      <div className="documents-list-skeleton" />
    </div>
  );
}
```

## ページ間遷移の最適化

```tsx
// app/(dashboard)/projects/page.tsx
import { useRouter } from 'next/navigation';

export default function Projects() {
  const router = useRouter();
  
  // プロジェクト選択時のナビゲーション
  const handleProjectSelect = (projectId: string) => {
    router.push(`/projects/${projectId}`);
  };
  
  return (
    // ...
  );
}
```

## まとめ

App Routerを活用することで、以下のメリットがあります：

- 階層的なレイアウト構造による効率的なUI設計
- サーバーコンポーネントによるパフォーマンス最適化
- 並行ルートによる複雑なUIレイアウトの実現
- 動的ルーティングによる柔軟なURL構造

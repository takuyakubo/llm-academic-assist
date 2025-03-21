# Next.js App Router 実装ガイド: LLM学術文書支援システム

## 1. Next.js App Router の基本概念

App Routerは、Next.js 13以降で導入された新しいルーティングパラダイムで、ファイルシステムベースのルーティングに加え、Reactの最新機能（Server Components、Streaming、Suspenseなど）を活用したアプリケーション構築をサポートします。

### 1.1 主要なファイル規約

| ファイル名 | 用途 | 対応する内容 |
|-----------|------|------------|
| `page.tsx` | ルートの公開インターフェース | URLにアクセスしたときに表示されるページコンポーネント |
| `layout.tsx` | 共有レイアウト | 複数のページで共有されるレイアウト（ネスト可能） |
| `loading.tsx` | ローディング状態 | ページ読み込み中に表示されるローディングUI |
| `error.tsx` | エラーハンドリング | エラー発生時に表示されるフォールバックUI |
| `not-found.tsx` | 404ページ | リソースが見つからない場合に表示されるページ |
| `route.ts` | APIエンドポイント | HTTP APIのサーバーサイド実装 |

## 2. プロジェクト初期セットアップ

### 2.1 プロジェクト作成

```bash
# App Routerを使用したNext.jsプロジェクトの作成
npx create-next-app@latest llm-academic-assist --typescript --eslint --tailwind --app
```

### 2.2 基本設定

```typescript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  // 画像最適化の設定
  images: {
    domains: ['storage.googleapis.com'],
    formats: ['image/avif', 'image/webp'],
  },
  // 国際化設定
  i18n: {
    locales: ['ja', 'en'],
    defaultLocale: 'ja',
  },
  // 実験的機能
  experimental: {
    serverActions: true,
    serverComponentsExternalPackages: ['pdf-parse', 'sharp'],
  },
};

module.exports = nextConfig;
```

## 3. ルート構造の実装

### 3.1 グローバルレイアウト

```tsx
// app/layout.tsx
import { Providers } from '@/components/providers';
import { Toaster } from '@/components/ui/toaster';
import "@/styles/globals.css";

export const metadata = {
  title: 'LLM学術文書支援システム',
  description: 'LLMを活用した学術文書の理解と学習を支援するプラットフォーム',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ja" suppressHydrationWarning>
      <body className="min-h-screen bg-background font-sans antialiased">
        <Providers>
          <div className="relative flex min-h-screen flex-col">
            {children}
          </div>
          <Toaster />
        </Providers>
      </body>
    </html>
  );
}
```

### 3.2 認証ルートグループ

```tsx
// app/(auth)/layout.tsx
export default function AuthLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex min-h-screen flex-col items-center justify-center py-12">
      <div className="w-full max-w-md space-y-8 px-8">
        <div className="flex flex-col items-center space-y-2 text-center">
          <h1 className="text-2xl font-bold">LLM学術文書支援システム</h1>
          <p className="text-muted-foreground">学術文書の理解と学習をサポート</p>
        </div>
        {children}
      </div>
    </div>
  );
}
```

### 3.3 ダッシュボードレイアウト

```tsx
// app/(dashboard)/layout.tsx
import { DashboardNav } from '@/components/dashboard/dashboard-nav';
import { SiteHeader } from '@/components/common/site-header';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth';
import { redirect } from 'next/navigation';

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const session = await getServerSession(authOptions);
  
  if (!session) {
    redirect('/login');
  }
  
  return (
    <div className="flex min-h-screen flex-col">
      <SiteHeader user={session.user} />
      <div className="container flex-1 grid grid-cols-1 gap-12 md:grid-cols-[200px_1fr] lg:grid-cols-[250px_1fr] py-10">
        <aside className="hidden md:block">
          <DashboardNav />
        </aside>
        <main>{children}</main>
      </div>
    </div>
  );
}
```

## 4. データフェッチング実装

App Router環境では、サーバーコンポーネント内でのデータフェッチングとクライアントコンポーネント内でのデータフェッチングを適切に使い分けることが重要です。

### 4.1 サーバーコンポーネントでのデータフェッチング

```tsx
// app/projects/page.tsx
import { ProjectList } from '@/components/projects/project-list';
import { fetchProjects } from '@/lib/data-fetching/projects';

export const metadata = {
  title: 'プロジェクト一覧 | LLM学術文書支援システム',
};

export default async function ProjectsPage() {
  // サーバーコンポーネント内で直接非同期データフェッチ
  const projects = await fetchProjects();
  
  return (
    <div className="space-y-6">
      <h1 className="text-3xl font-bold">プロジェクト一覧</h1>
      <ProjectList initialProjects={projects} />
    </div>
  );
}
```

### 4.2 クライアントコンポーネントでのデータフェッチング

```tsx
// components/projects/project-list.tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { useQuery } from '@tanstack/react-query';
import { ProjectCard } from '@/components/projects/project-card';
import { Button } from '@/components/ui/button';
import { Project } from '@/types/project';

async function fetchProjects() {
  const res = await fetch('/api/projects');
  if (!res.ok) {
    throw new Error('プロジェクトの取得に失敗しました');
  }
  return res.json();
}

export function ProjectList({ initialProjects }: { initialProjects: Project[] }) {
  const router = useRouter();
  
  // React QueryでAPIからデータをフェッチ（初期データはサーバーから提供）
  const { data: projects, isLoading, error } = useQuery({
    queryKey: ['projects'],
    queryFn: fetchProjects,
    initialData: initialProjects,
    refetchOnWindowFocus: false,
  });
  
  if (isLoading && !initialProjects.length) {
    return <div>読み込み中...</div>;
  }
  
  if (error) {
    return <div>エラーが発生しました: {error.message}</div>;
  }
  
  return (
    <div className="space-y-4">
      <div className="flex justify-between items-center">
        <h2 className="text-xl font-semibold">
          {projects.length > 0
            ? `${projects.length}件のプロジェクト`
            : 'プロジェクトがありません'}
        </h2>
        <Button onClick={() => router.push('/projects/new')}>
          新規プロジェクト
        </Button>
      </div>
      
      <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">
        {projects.map((project) => (
          <ProjectCard key={project.id} project={project} />
        ))}
      </div>
    </div>
  );
}
```

### 4.3 APIルートの実装

```tsx
// app/api/projects/route.ts
import { NextResponse } from 'next/server';
import { getServerSession } from 'next-auth/next';
import { authOptions } from '@/lib/auth';
import { z } from 'zod';

// リクエストバリデーションスキーマ
const projectSchema = z.object({
  name: z.string().min(1),
  description: z.string().optional(),
});

export async function GET() {
  const session = await getServerSession(authOptions);
  
  if (!session) {
    return NextResponse.json({ error: '認証が必要です' }, { status: 401 });
  }
  
  try {
    // バックエンドAPIへのリクエスト
    const response = await fetch(`${process.env.API_URL}/projects`, {
      headers: {
        'Authorization': `Bearer ${session.accessToken}`,
      },
    });
    
    if (!response.ok) {
      const error = await response.json();
      return NextResponse.json({ error: error.message || 'サーバーエラー' }, { status: response.status });
    }
    
    const data = await response.json();
    return NextResponse.json(data);
  } catch (error) {
    return NextResponse.json({ error: '内部サーバーエラー' }, { status: 500 });
  }
}

export async function POST(request: Request) {
  try {
    const session = await getServerSession(authOptions);
    
    if (!session) {
      return NextResponse.json({ error: '認証が必要です' }, { status: 401 });
    }
    
    const body = await request.json();
    const validatedData = projectSchema.parse(body);
    
    // バックエンドAPIへのリクエスト
    const response = await fetch(`${process.env.API_URL}/projects`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${session.accessToken}`,
      },
      body: JSON.stringify({
        ...validatedData,
        userId: session.user.id,
      }),
    });
    
    if (!response.ok) {
      const error = await response.json();
      return NextResponse.json({ error: error.message || 'サーバーエラー' }, { status: response.status });
    }
    
    const data = await response.json();
    return NextResponse.json(data);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json({ error: error.errors }, { status: 400 });
    }
    
    return NextResponse.json({ error: '内部サーバーエラー' }, { status: 500 });
  }
}
```

## 5. サーバーアクション (Server Actions) の活用

Next.js 14からは、フォーム処理のためのServer Actionsが安定版として提供されています。この機能を使って、クライアントコンポーネントからサーバーサイドの処理を直接呼び出すことができます。

### 5.1 サーバーアクションの定義

```tsx
// app/actions/document-actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { getServerSession } from 'next-auth/next';
import { authOptions } from '@/lib/auth';
import { z } from 'zod';

const documentSchema = z.object({
  title: z.string().min(1, 'タイトルは必須です'),
  projectId: z.string().min(1, 'プロジェクトIDは必須です'),
  file: z.any().optional(),
});

export async function uploadDocument(formData: FormData) {
  const session = await getServerSession(authOptions);
  
  if (!session) {
    throw new Error('認証が必要です');
  }
  
  const title = formData.get('title') as string;
  const projectId = formData.get('projectId') as string;
  const file = formData.get('file') as File;
  
  try {
    // バリデーション
    documentSchema.parse({ title, projectId, file });
    
    if (!file) {
      throw new Error('ファイルは必須です');
    }
    
    // ファイルをサーバーにアップロード
    const fileData = new FormData();
    fileData.append('file', file);
    fileData.append('title', title);
    fileData.append('projectId', projectId);
    
    const response = await fetch(`${process.env.API_URL}/documents/upload`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${session.accessToken}`,
      },
      body: fileData,
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message || 'ファイルのアップロードに失敗しました');
    }
    
    // キャッシュを更新して最新データを反映
    revalidatePath(`/projects/${projectId}/documents`);
    
    // 成功したらドキュメント一覧ページにリダイレクト
    redirect(`/projects/${projectId}/documents`);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return { success: false, error: error.errors };
    }
    
    return { 
      success: false, 
      error: error instanceof Error ? error.message : '不明なエラーが発生しました' 
    };
  }
}
```

### 5.2 サーバーアクションをフォームで使用

```tsx
// components/documents/document-upload-form.tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { toast } from '@/components/ui/use-toast';
import { uploadDocument } from '@/app/actions/document-actions';

export function DocumentUploadForm({ projectId }: { projectId: string }) {
  const [isPending, setIsPending] = useState(false);
  const router = useRouter();
  
  async function onSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();
    setIsPending(true);
    
    try {
      const formData = new FormData(event.currentTarget);
      formData.append('projectId', projectId);
      
      const result = await uploadDocument(formData);
      
      if (result && !result.success) {
        toast({
          title: 'エラー',
          description: result.error || '文書のアップロードに失敗しました',
          variant: 'destructive',
        });
      } else {
        toast({
          title: '文書をアップロードしました',
          description: '文書の処理を開始します',
        });
      }
    } catch (error) {
      toast({
        title: 'エラー',
        description: error instanceof Error ? error.message : '不明なエラーが発生しました',
        variant: 'destructive',
      });
    } finally {
      setIsPending(false);
    }
  }
  
  return (
    <form onSubmit={onSubmit} className="space-y-6">
      <div className="space-y-2">
        <Label htmlFor="title">タイトル</Label>
        <Input id="title" name="title" required placeholder="文書のタイトル" />
      </div>
      
      <div className="space-y-2">
        <Label htmlFor="file">ファイル（PDF/画像）</Label>
        <Input id="file" name="file" type="file" accept=".pdf,.jpg,.jpeg,.png" required />
      </div>
      
      <div className="flex items-center gap-4">
        <Button type="submit" disabled={isPending}>
          {isPending ? 'アップロード中...' : 'アップロード'}
        </Button>
        <Button 
          type="button" 
          variant="outline" 
          onClick={() => router.back()}
          disabled={isPending}
        >
          キャンセル
        </Button>
      </div>
    </form>
  );
}
```

## 6. 動的ルートとパラメータの処理

Next.jsのApp Routerでは、動的ルートを使用して可変パラメータを扱うことができます。

### 6.1 動的ルートの実装

```tsx
// app/projects/[id]/page.tsx
import { notFound } from 'next/navigation';
import { fetchProject } from '@/lib/data-fetching/projects';
import { ProjectDetail } from '@/components/projects/project-detail';

export async function generateMetadata({ params }: { params: { id: string } }) {
  const project = await fetchProject(params.id);
  
  if (!project) {
    return {
      title: 'プロジェクトが見つかりません',
    };
  }
  
  return {
    title: `${project.name} | LLM学術文書支援システム`,
    description: project.description,
  };
}

export default async function ProjectPage({ params }: { params: { id: string } }) {
  const project = await fetchProject(params.id);
  
  if (!project) {
    notFound();
  }
  
  return <ProjectDetail project={project} />;
}
```

### 6.2 パラレルルートとインターセプトルート

App Routerの高度な機能として、パラレルルート（同時に複数のコンテンツを表示）とインターセプトルート（現在のページを維持しながら別ルートの内容を表示）があります。

```tsx
// app/projects/[id]/@documents/page.tsx (パラレルルート)
import { DocumentList } from '@/components/documents/document-list';
import { fetchDocumentsByProject } from '@/lib/data-fetching/documents';

export default async function DocumentsTab({ params }: { params: { id: string } }) {
  const documents = await fetchDocumentsByProject(params.id);
  
  return <DocumentList documents={documents} projectId={params.id} />;
}

// app/projects/[id]/layout.tsx (パラレルルートのレイアウト)
export default function ProjectLayout({
  children,
  documents,
  chat,
}: {
  children: React.ReactNode;
  documents: React.ReactNode;
  chat: React.ReactNode;
}) {
  return (
    <div className="space-y-6">
      <div className="project-main">
        {children}
      </div>
      <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
        <div className="documents-section">
          <h2 className="text-xl font-bold mb-4">文書一覧</h2>
          {documents}
        </div>
        <div className="chat-section">
          <h2 className="text-xl font-bold mb-4">対話履歴</h2>
          {chat}
        </div>
      </div>
    </div>
  );
}

// app/projects/[id]/document/[docId]/(..)chat/[chatId]/page.tsx (インターセプトルート)
import { Modal } from '@/components/ui/modal';
import { ChatInterface } from '@/components/chat/chat-interface';
import { fetchChat } from '@/lib/data-fetching/chat';

export default async function InterceptedChatPage({
  params
}: {
  params: { id: string, docId: string, chatId: string }
}) {
  const chat = await fetchChat(params.chatId);
  
  return (
    <Modal>
      <ChatInterface 
        initialMessages={chat.messages} 
        projectId={params.id} 
        documentId={params.docId} 
        chatId={params.chatId} 
      />
    </Modal>
  );
}
```

# Next.js App Router パフォーマンス最適化

## 概要

Next.js 14のApp Routerを使った学術文書支援システムのパフォーマンスを最適化するための手法とベストプラクティスについて説明します。

## サーバーコンポーネントの活用

App Routerではデフォルトでサーバーコンポーネントが使用されます。サーバーコンポーネントをうまく活用することでパフォーマンスを向上させることができます。

```tsx
// app/(dashboard)/projects/[id]/page.tsx
import { getProjectDetails } from '@/lib/db';

// サーバーコンポーネント - データフェッチをサーバー側で行う
export default async function ProjectPage({ params }: { params: { id: string } }) {
  // サーバー側でデータ取得
  const project = await getProjectDetails(params.id);
  
  return (
    <div>
      <h1>{project.name}</h1>
      <p>{project.description}</p>
      <DocumentList documents={project.documents} />
    </div>
  );
}
```

### クライアントコンポーネントの限定使用

インタラクティブな部分のみをクライアントコンポーネントとして実装します。

```tsx
// components/document-chat.tsx
'use client';

import { useState } from 'react';

export default function DocumentChat({ documentId }: { documentId: string }) {
  const [message, setMessage] = useState('');
  const [chatHistory, setChatHistory] = useState([]);
  
  // インタラクティブな処理をクライアント側で実行
  const handleSubmit = async (e) => {
    e.preventDefault();
    // ...
  };
  
  return (
    <div className="chat-container">
      {/* チャットUIコンポーネント */}
    </div>
  );
}
```

## ルートセグメントレベルでのキャッシング

App Routerではルートセグメントレベルでのキャッシュ制御が可能です。

```tsx
// app/(dashboard)/projects/page.tsx
import { getProjects } from '@/lib/db';

// デフォルトではキャッシュが有効
export default async function ProjectsPage() {
  const projects = await getProjects();
  return (
    <div>{/* Projects list */}</div>
  );
}
```

```tsx
// app/(dashboard)/projects/[id]/documents/[docId]/page.tsx
import { getDocumentContent } from '@/lib/db';

// ダイナミックセグメントを含むページでキャッシュを無効化する場合
export const dynamic = 'force-dynamic';
// または特定の期間だけキャッシュする場合
export const revalidate = 3600; // 1時間ごとに再検証

export default async function DocumentPage({ params }: { params: { docId: string } }) {
  const document = await getDocumentContent(params.docId);
  return (
    <div>{/* Document content */}</div>
  );
}
```

## Suspenseによる段階的ロード

コンテンツの一部をSuspenseで包み、残りのUIをブロックせずに表示することができます。

```tsx
// app/(dashboard)/projects/[id]/page.tsx
import { Suspense } from 'react';
import { DocumentSkeleton } from '@/components/skeletons';

export default function ProjectPage({ params }: { params: { id: string } }) {
  return (
    <div className="project-container">
      <ProjectHeader id={params.id} />
      
      <div className="documents-container">
        <h2>プロジェクト文書</h2>
        <Suspense fallback={<DocumentSkeleton count={3} />}>
          {/* データフェッチが完了するまでスケルトン表示 */}
          <DocumentList projectId={params.id} />
        </Suspense>
      </div>
      
      <div className="notes-container">
        <h2>学習ノート</h2>
        <Suspense fallback={<p>ノートを読み込み中...</p>}>
          <NotesList projectId={params.id} />
        </Suspense>
      </div>
    </div>
  );
}
```

## ストリーミングレスポンスの最適化

LLMからのストリーミングレスポンスを効率的に処理するためのパターン：

```tsx
// app/api/chat/route.ts
import { LLMStream } from '@/lib/llm/stream';

export async function POST(req: Request) {
  const { documentId, message } = await req.json();
  
  // ストリーミングの設定
  const stream = new LLMStream();
  
  // ヘッダーを設定
  const headers = {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache, no-transform',
    'Connection': 'keep-alive',
  };
  
  try {
    // ストリームレスポンスを開始
    const response = new Response(stream, { headers });
    
    // バックグラウンドでストリームを処理
    streamLLMResponse(documentId, message, stream).catch(console.error);
    
    return response;
  } catch (error) {
    return new Response(JSON.stringify({ error: 'ストリーミング中にエラーが発生しました' }), {
      status: 500,
      headers: { 'Content-Type': 'application/json' },
    });
  }
}

async function streamLLMResponse(documentId: string, message: string, stream: LLMStream) {
  try {
    // ドキュメントコンテキストの取得
    const document = await getDocumentContent(documentId);
    
    // LLM ストリーミング処理
    // ...
    
    // 最後にストリームを閉じる
    stream.close();
  } catch (error) {
    stream.error(error);
  }
}
```

## 画像最適化

Next.jsの`Image`コンポーネントを使用して画像の最適化を行います。

```tsx
// components/document-thumbnail.tsx
import Image from 'next/image';

export default function DocumentThumbnail({ document }) {
  return (
    <div className="thumbnail-container">
      <Image
        src={document.thumbnailUrl}
        alt={document.title}
        width={240}
        height={320}
        quality={80}
        priority={false}
        loading="lazy"
        placeholder="blur"
        blurDataURL="data:image/svg+xml;base64,..."
      />
    </div>
  );
}
```

## フォント最適化

ページロード体験を向上させるためにフォントの読み込みを最適化します。

```tsx
// app/layout.tsx
import { Inter, Noto_Sans_JP } from 'next/font/google';

// フォントの最適化設定
const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
});

const notoSansJP = Noto_Sans_JP({
  subsets: ['latin'],
  weight: ['400', '500', '700'],
  display: 'swap',
  variable: '--font-noto-sans-jp',
});

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="ja" className={`${inter.variable} ${notoSansJP.variable}`}>
      <body>
        {children}
      </body>
    </html>
  );
}
```

## バンドルサイズの最適化

### コード分割

動的インポートでコードを必要に応じて読み込みます。

```tsx
// app/(dashboard)/projects/[id]/documents/[docId]/page.tsx
import dynamic from 'next/dynamic';

// 数式レンダリングコンポーネントを動的インポート
const MathRenderer = dynamic(() => import('@/components/math-renderer'), {
  loading: () => <p>数式を読み込み中...</p>,
  ssr: false, // クライアントサイドでのみレンダリング
});

export default function DocumentPage({ params }: { params: { docId: string } }) {
  // ...
  
  return (
    <div>
      <h1>{document.title}</h1>
      <div className="document-content">
        {document.hasComplexMath ? (
          <MathRenderer content={document.content} />
        ) : (
          <MarkdownRenderer content={document.content} />
        )}
      </div>
    </div>
  );
}
```

## Server Actionsの最適化

サーバーアクションを最適化して、効率的なフォーム処理を実現します。

```tsx
// app/actions/document.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createDocument(formData: FormData) {
  // フォームデータから必要な情報を抽出
  const title = formData.get('title') as string;
  const file = formData.get('file') as File;
  const projectId = formData.get('projectId') as string;
  
  // ファイルアップロード処理
  const documentId = await uploadAndProcessDocument(file, title, projectId);
  
  // 関連するパスのキャッシュを無効化
  revalidatePath(`/projects/${projectId}`);
  
  // 新しいドキュメントページにリダイレクト
  redirect(`/projects/${projectId}/documents/${documentId}`);
}
```

## メモリ使用量の最適化

大きなデータセットを扱う場合のメモリ最適化テクニック：

```typescript
// lib/document-processor.ts

// データをチャンクに分割して処理する
export async function processLargeDocument(documentId: string) {
  const document = await getDocumentMetadata(documentId);
  const pageCount = document.pageCount;
  
  // 一度に処理するページ数
  const CHUNK_SIZE = 5;
  
  for (let startPage = 1; startPage <= pageCount; startPage += CHUNK_SIZE) {
    const endPage = Math.min(startPage + CHUNK_SIZE - 1, pageCount);
    
    // ページ範囲を処理
    await processDocumentRange(documentId, startPage, endPage);
    
    // 処理状況を更新
    await updateProcessingStatus(documentId, {
      processedPages: endPage,
      totalPages: pageCount,
    });
  }
  
  return { success: true, pageCount };
}
```

## エッジランタイムの活用

リアルタイム処理が必要なAPIルートでは、エッジランタイムを活用します。

```typescript
// app/api/chat/route.ts
export const runtime = 'edge';

export async function POST(req: Request) {
  // ...
}
```

## プログラマティックなプリフェッチ

予測可能なユーザーアクション（例：プロジェクトの選択）に対するプリフェッチを実装します。

```tsx
// components/projects-list.tsx
'use client';

import { useRouter } from 'next/navigation';
import { useEffect } from 'react';

export default function ProjectsList({ projects }) {
  const router = useRouter();
  
  useEffect(() => {
    // 最近アクセスした上位3つのプロジェクトをプリフェッチ
    const recentProjects = projects.slice(0, 3);
    
    for (const project of recentProjects) {
      router.prefetch(`/projects/${project.id}`);
    }
  }, [projects, router]);
  
  return (
    <ul className="projects-list">
      {projects.map(project => (
        <li key={project.id}>
          <a
            href={`/projects/${project.id}`}
            onMouseEnter={() => router.prefetch(`/projects/${project.id}`)}
          >
            {project.name}
          </a>
        </li>
      ))}
    </ul>
  );
}
```

## App Router特有の最適化テクニック

### Parallel Routesの活用

並行ルートを使って、ページの異なる部分を独立してロードします。

```tsx
// app/(dashboard)/projects/[id]/documents/[docId]/layout.tsx
export default function DocumentLayout({
  children,
  chat,
}: {
  children: React.ReactNode;
  chat: React.ReactNode;
}) {
  return (
    <div className="document-layout">
      <div className="document-content">
        {children}
      </div>
      <div className="document-chat">
        {chat}
      </div>
    </div>
  );
}
```

フォルダ構造：
```
app/(dashboard)/projects/[id]/documents/[docId]/
├── page.tsx               # 文書表示コンポーネント
└── @chat/                 # チャット用並行ルート
    └── page.tsx           # チャットコンポーネント
```

### Route Interceptionsの活用

モーダルやポップオーバーUIを実装するためのルートインターセプション：

```
app/(dashboard)/projects/
├── page.tsx               # プロジェクト一覧
└── (.)new/                # インターセプトルート（モーダル）
    └── page.tsx           # 新規プロジェクト作成フォーム
```

```tsx
// app/(dashboard)/projects/(.)new/page.tsx
import { Modal } from '@/components/ui/modal';
import ProjectForm from '@/components/project-form';

export default function NewProjectModal() {
  return (
    <Modal>
      <h2>新規プロジェクト作成</h2>
      <ProjectForm />
    </Modal>
  );
}
```

## データ取得の最適化

### React Queryの活用

クライアントサイドでのデータ取得とキャッシュには React Query を活用します。

```tsx
// providers/query-provider.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // 1分間はデータを新鮮と見なす
        retry: 1,
        refetchOnWindowFocus: false,
      },
    },
  }));

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === 'development' && <ReactQueryDevtools />}
    </QueryClientProvider>
  );
}
```

```tsx
// hooks/use-document.ts
'use client';

import { useQuery } from '@tanstack/react-query';

export function useDocument(documentId: string) {
  return useQuery({
    queryKey: ['document', documentId],
    queryFn: async () => {
      const res = await fetch(`/api/documents/${documentId}`);
      if (!res.ok) throw new Error('文書の取得に失敗しました');
      return res.json();
    },
  });
}
```

## パフォーマンスモニタリング

アプリケーションのパフォーマンスを継続的にモニタリングするための仕組み：

```tsx
// instrumentation.ts
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    const { setupMonitoring } = await import('./lib/monitoring/server');
    setupMonitoring();
  } else {
    const { setupBrowserMonitoring } = await import('./lib/monitoring/client');
    setupBrowserMonitoring();
  }
}
```

## まとめ

Next.js App Routerを使用したLLM学術文書支援システムのパフォーマンス最適化のポイント：

1. **サーバーコンポーネントの積極的活用**：データフェッチと初期レンダリングをサーバーで行う
2. **クライアントコンポーネントの最小化**：インタラクティブな部分のみクライアントコンポーネントとする
3. **Suspenseとストリーミングの活用**：段階的な読み込みでユーザー体験を向上
4. **適切なキャッシュ戦略**：ルートセグメントレベルでのキャッシュ制御
5. **コード分割とレイジーロード**：必要なコードのみを必要なタイミングで読み込む
6. **パラレルルートの活用**：独立した部分を並行して読み込む
7. **エッジランタイムの活用**：低レイテンシーを実現する
8. **最適なデータフェッチ**：オーバーフェッチを避け、必要なデータのみを取得する
9. **定期的なパフォーマンス計測とチューニング**：パフォーマンスメトリクスを測定し継続改善

これらの最適化テクニックを組み合わせることで、大規模な学術文書や複雑な数式処理、LLMとの対話を含むシステムでも、高速で応答性の高いユーザー体験を提供できます。

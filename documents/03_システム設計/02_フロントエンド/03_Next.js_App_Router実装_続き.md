# Next.js App Router 実装ガイド（続き）: LLM学術文書支援システム

## 7. ストリーミングレスポンスの実装

LLMからのレスポンスはストリーミング形式で取得することが一般的です。Next.jsのApp Routerでは、React Suspenseを利用してストリーミングレスポンスを実装できます。

### 7.1 ストリーミングAPIエンドポイント

```tsx
// app/api/chat/route.ts
import { NextResponse } from 'next/server';
import { getServerSession } from 'next-auth/next';
import { authOptions } from '@/lib/auth';

export async function POST(req: Request) {
  const session = await getServerSession(authOptions);
  
  if (!session) {
    return NextResponse.json({ error: '認証が必要です' }, { status: 401 });
  }
  
  const { messages, projectId, documentId } = await req.json();
  
  // コンテンツタイプをtext/event-streamに設定
  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    async start(controller) {
      try {
        // バックエンドAPIにリクエストを送信
        const response = await fetch(`${process.env.API_URL}/chat/stream`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${session.accessToken}`,
          },
          body: JSON.stringify({
            messages,
            projectId,
            documentId,
            userId: session.user.id,
          }),
        });
        
        if (!response.ok) {
          const error = await response.json();
          controller.enqueue(encoder.encode(JSON.stringify({ error: error.message || 'エラーが発生しました' })));
          controller.close();
          return;
        }
        
        // バックエンドからのストリームを読み込み
        const reader = response.body?.getReader();
        if (!reader) {
          controller.enqueue(encoder.encode(JSON.stringify({ error: 'ストリームの初期化に失敗しました' })));
          controller.close();
          return;
        }
        
        while (true) {
          const { done, value } = await reader.read();
          if (done) break;
          controller.enqueue(value);
        }
        controller.close();
      } catch (error) {
        controller.enqueue(encoder.encode(JSON.stringify({ 
          error: error instanceof Error ? error.message : '不明なエラーが発生しました' 
        })));
        controller.close();
      }
    }
  });
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache, no-transform',
      'Connection': 'keep-alive',
    },
  });
}
```

### 7.2 フロントエンドでのストリーミング表示

```tsx
// components/chat/chat-interface.tsx
'use client';

import { useEffect, useState, useRef } from 'react';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { ChatMessage } from '@/components/chat/chat-message';
import { toast } from '@/components/ui/use-toast';

type Message = {
  role: 'user' | 'assistant' | 'system';
  content: string;
};

export function ChatInterface({
  initialMessages = [],
  projectId,
  documentId,
  chatId,
}: {
  initialMessages?: Message[];
  projectId: string;
  documentId?: string;
  chatId?: string;
}) {
  const [messages, setMessages] = useState<Message[]>(initialMessages);
  const [input, setInput] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [streamedResponse, setStreamedResponse] = useState('');
  const messagesEndRef = useRef<HTMLDivElement>(null);
  
  // メッセージリストの最下部に自動スクロール
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages, streamedResponse]);
  
  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (!input.trim() || isLoading) return;
    
    const userMessage = { role: 'user' as const, content: input };
    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setIsLoading(true);
    setStreamedResponse('');
    
    try {
      // APIエンドポイントにリクエスト
      const response = await fetch('/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          messages: [...messages, userMessage],
          projectId,
          documentId,
          chatId,
        }),
      });
      
      if (!response.ok) {
        throw new Error('AIレスポンスの生成に失敗しました');
      }
      
      // ストリームの読み込み
      const reader = response.body?.getReader();
      if (!reader) throw new Error('レスポンスの読み取りに失敗しました');
      
      const decoder = new TextDecoder();
      let responseText = '';
      
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        
        const chunk = decoder.decode(value, { stream: true });
        responseText += chunk;
        setStreamedResponse(responseText);
      }
      
      // ストリーミング完了後にメッセージリストに追加
      setMessages(prev => [...prev, { role: 'assistant', content: responseText }]);
    } catch (error) {
      toast({
        title: 'エラーが発生しました',
        description: error instanceof Error ? error.message : '不明なエラー',
        variant: 'destructive',
      });
    } finally {
      setIsLoading(false);
      setStreamedResponse('');
    }
  }
  
  return (
    <div className="flex flex-col h-full">
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((message, i) => (
          <ChatMessage key={i} message={message} />
        ))}
        
        {streamedResponse && (
          <ChatMessage message={{ role: 'assistant', content: streamedResponse }} isStreaming />
        )}
        
        <div ref={messagesEndRef} />
      </div>
      
      <form onSubmit={handleSubmit} className="border-t p-4">
        <div className="flex space-x-2">
          <Textarea
            value={input}
            onChange={e => setInput(e.target.value)}
            placeholder="メッセージを入力..."
            className="flex-1 min-h-[80px]"
            disabled={isLoading}
          />
          <Button type="submit" disabled={isLoading || !input.trim()}>
            {isLoading ? '送信中...' : '送信'}
          </Button>
        </div>
      </form>
    </div>
  );
}
```

### 7.3 React Suspenseによるストリーミングの最適化

```tsx
// app/projects/[id]/documents/[docId]/page.tsx
import { Suspense } from 'react';
import { fetchDocument } from '@/lib/data-fetching/documents';
import { DocumentViewer } from '@/components/documents/document-viewer';
import { ChatContainer } from '@/components/chat/chat-container';
import { LoadingChatUI } from '@/components/chat/loading-chat-ui';
import { notFound } from 'next/navigation';

export default async function DocumentPage({
  params,
}: {
  params: { id: string; docId: string };
}) {
  const document = await fetchDocument(params.docId);
  
  if (!document) {
    notFound();
  }
  
  return (
    <div className="grid grid-cols-1 lg:grid-cols-2 gap-6 h-[calc(100vh-8rem)]">
      <div className="h-full overflow-hidden border rounded-lg">
        <DocumentViewer document={document} />
      </div>
      
      <div className="h-full border rounded-lg flex flex-col">
        <div className="p-4 border-b">
          <h2 className="text-xl font-bold">文書に関する質問</h2>
          <p className="text-sm text-muted-foreground">
            この文書の内容について質問できます
          </p>
        </div>
        
        {/* Suspenseを使用して重いコンポーネントをラップ */}
        <Suspense fallback={<LoadingChatUI />}>
          <ChatContainer 
            projectId={params.id} 
            documentId={params.docId} 
          />
        </Suspense>
      </div>
    </div>
  );
}

// components/chat/chat-container.tsx
import { fetchChatHistory } from '@/lib/data-fetching/chat';
import { ChatInterface } from '@/components/chat/chat-interface';

export async function ChatContainer({
  projectId,
  documentId,
}: {
  projectId: string;
  documentId: string;
}) {
  // このコンポーネントの初期レンダリング中にデータをフェッチ
  // Suspenseによってローディング状態が表示される
  const chatHistory = await fetchChatHistory(projectId, documentId);
  
  return (
    <ChatInterface
      initialMessages={chatHistory.messages}
      projectId={projectId}
      documentId={documentId}
      chatId={chatHistory.id}
    />
  );
}
```

## 8. エラーハンドリングとローディング状態

App Routerでは、エラーハンドリングとローディング状態の管理が簡素化されています。

### 8.1 エラーハンドリング

```tsx
// app/projects/[id]/error.tsx
'use client';

import { useEffect } from 'react';
import { Button } from '@/components/ui/button';

export default function ProjectError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // エラーをロギングサービスに送信
    console.error(error);
  }, [error]);
  
  return (
    <div className="flex flex-col items-center justify-center min-h-[400px] p-6 text-center">
      <h2 className="text-2xl font-bold mb-4">エラーが発生しました</h2>
      <p className="text-muted-foreground mb-6">
        {error.message || 'プロジェクトの読み込み中に問題が発生しました'}
      </p>
      <Button onClick={() => reset()}>再試行</Button>
    </div>
  );
}
```

### 8.2 ローディング状態

```tsx
// app/projects/[id]/loading.tsx
import { Skeleton } from '@/components/ui/skeleton';

export default function ProjectLoading() {
  return (
    <div className="space-y-6">
      <div className="space-y-2">
        <Skeleton className="h-10 w-[250px]" />
        <Skeleton className="h-4 w-[350px]" />
      </div>
      
      <div className="grid gap-6 grid-cols-1 md:grid-cols-2">
        <div className="space-y-4">
          <Skeleton className="h-8 w-[120px]" />
          <div className="space-y-2">
            <Skeleton className="h-16 w-full" />
            <Skeleton className="h-16 w-full" />
            <Skeleton className="h-16 w-full" />
          </div>
        </div>
        
        <div className="space-y-4">
          <Skeleton className="h-8 w-[100px]" />
          <div className="space-y-2">
            <Skeleton className="h-32 w-full" />
            <Skeleton className="h-32 w-full" />
          </div>
        </div>
      </div>
    </div>
  );
}
```

### 8.3 notFound ページ

```tsx
// app/projects/[id]/not-found.tsx
import Link from 'next/link';
import { Button } from '@/components/ui/button';

export default function ProjectNotFound() {
  return (
    <div className="flex flex-col items-center justify-center min-h-[400px] p-6 text-center">
      <h2 className="text-2xl font-bold mb-4">プロジェクトが見つかりません</h2>
      <p className="text-muted-foreground mb-6">
        指定されたプロジェクトは削除されたか、存在しません。
      </p>
      <Button asChild>
        <Link href="/projects">プロジェクト一覧に戻る</Link>
      </Button>
    </div>
  );
}
```

## 9. 認証とセッション管理

NextAuth.js（現在はAuth.js）を使用して認証機能を実装します。

### 9.1 認証設定

```tsx
// lib/auth.ts
import { NextAuthOptions } from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import { z } from 'zod';

// 認証情報のバリデーションスキーマ
const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export const authOptions: NextAuthOptions = {
  providers: [
    CredentialsProvider({
      name: 'Credentials',
      credentials: {
        email: { label: 'メールアドレス', type: 'email' },
        password: { label: 'パスワード', type: 'password' },
      },
      async authorize(credentials) {
        try {
          // バリデーション
          const { email, password } = loginSchema.parse(credentials);
          
          // バックエンドAPIでの認証
          const response = await fetch(`${process.env.API_URL}/auth/login`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ email, password }),
          });
          
          if (!response.ok) {
            return null;
          }
          
          const user = await response.json();
          return user;
        } catch (error) {
          return null;
        }
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      // 初回ログイン時にユーザー情報をトークンに追加
      if (user) {
        token.id = user.id;
        token.accessToken = user.accessToken;
      }
      return token;
    },
    async session({ session, token }) {
      // セッションにユーザー情報を追加
      if (token) {
        session.user.id = token.id as string;
        session.accessToken = token.accessToken as string;
      }
      return session;
    },
  },
  pages: {
    signIn: '/login',
    error: '/login',
  },
  session: {
    strategy: 'jwt',
    maxAge: 30 * 24 * 60 * 60, // 30日
  },
  secret: process.env.NEXTAUTH_SECRET,
};
```

### 9.2 セッションプロバイダー

```tsx
// components/providers.tsx
'use client';

import { SessionProvider } from 'next-auth/react';
import { ThemeProvider } from 'next-themes';

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <SessionProvider>
      <ThemeProvider 
        attribute="class" 
        defaultTheme="system" 
        enableSystem
      >
        {children}
      </ThemeProvider>
    </SessionProvider>
  );
}
```

## 10. 数式と文書表示の実装

学術文書支援システムの重要な機能として、数式表示と文書ビューアーの実装が挙げられます。

### 10.1 KaTeXによる数式レンダリング

```tsx
// components/math/math-renderer.tsx
'use client';

import { useEffect, useRef } from 'react';
import katex from 'katex';
import 'katex/dist/katex.min.css';

interface MathRendererProps {
  math: string;
  displayMode?: boolean;
  className?: string;
}

export function MathRenderer({
  math,
  displayMode = false,
  className = '',
}: MathRendererProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    if (containerRef.current) {
      try {
        katex.render(math, containerRef.current, {
          displayMode,
          throwOnError: false,
          output: 'html',
          trust: true,
        });
      } catch (error) {
        console.error('数式のレンダリングに失敗:', error);
        containerRef.current.textContent = math;
      }
    }
  }, [math, displayMode]);
  
  return <div ref={containerRef} className={className} />;
}
```

### 10.2 PDF文書ビューアー

```tsx
// components/documents/document-viewer.tsx
'use client';

import { useState } from 'react';
import { Document, Page, pdfjs } from 'react-pdf';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Slider } from '@/components/ui/slider';
import { DocumentType } from '@/types/document';
import 'react-pdf/dist/esm/Page/AnnotationLayer.css';
import 'react-pdf/dist/esm/Page/TextLayer.css';

// PDF.jsワーカーの設定
pdfjs.GlobalWorkerOptions.workerSrc = `//cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjs.version}/pdf.worker.min.js`;

interface DocumentViewerProps {
  document: DocumentType;
}

export function DocumentViewer({ document }: DocumentViewerProps) {
  const [numPages, setNumPages] = useState<number | null>(null);
  const [pageNumber, setPageNumber] = useState<number>(1);
  const [scale, setScale] = useState<number>(1.0);
  
  function onDocumentLoadSuccess({ numPages }: { numPages: number }) {
    setNumPages(numPages);
  }
  
  function changePage(offset: number) {
    setPageNumber(prevPageNumber => {
      const newPageNumber = prevPageNumber + offset;
      return Math.max(1, Math.min(numPages || 1, newPageNumber));
    });
  }
  
  function onPageInputChange(e: React.ChangeEvent<HTMLInputElement>) {
    const value = parseInt(e.target.value);
    if (!isNaN(value) && value >= 1 && value <= (numPages || 1)) {
      setPageNumber(value);
    }
  }
  
  return (
    <div className="flex flex-col h-full">
      <div className="flex items-center justify-between p-2 border-b">
        <div className="flex items-center space-x-2">
          <Button
            size="sm"
            variant="outline"
            onClick={() => changePage(-1)}
            disabled={pageNumber <= 1}
          >
            前へ
          </Button>
          <div className="flex items-center space-x-2">
            <Input
              type="number"
              min={1}
              max={numPages || 1}
              value={pageNumber}
              onChange={onPageInputChange}
              className="w-16"
            />
            <span>/ {numPages || '-'}</span>
          </div>
          <Button
            size="sm"
            variant="outline"
            onClick={() => changePage(1)}
            disabled={pageNumber >= (numPages || 1)}
          >
            次へ
          </Button>
        </div>
        
        <div className="flex items-center space-x-2 w-1/3">
          <Button
            size="sm"
            variant="outline"
            onClick={() => setScale(s => Math.max(0.5, s - 0.1))}
          >
            -
          </Button>
          <Slider
            value={[scale * 100]}
            min={50}
            max={200}
            step={10}
            onValueChange={([value]) => setScale(value / 100)}
            className="w-full"
          />
          <Button
            size="sm"
            variant="outline"
            onClick={() => setScale(s => Math.min(2.0, s + 0.1))}
          >
            +
          </Button>
        </div>
      </div>
      
      <div className="flex-1 overflow-auto">
        <div className="flex justify-center min-h-full p-4">
          <Document
            file={document.fileUrl}
            onLoadSuccess={onDocumentLoadSuccess}
            loading={<div className="text-center py-10">PDFを読み込み中...</div>}
            error={<div className="text-center py-10 text-red-500">PDFの読み込みに失敗しました</div>}
          >
            <Page
              pageNumber={pageNumber}
              scale={scale}
              renderTextLayer
              renderAnnotationLayer
              loading={<div className="text-center py-10">ページを読み込み中...</div>}
            />
          </Document>
        </div>
      </div>
    </div>
  );
}
```

## 11. Next.js App Router のベストプラクティス

### 11.1 ディレクトリ構造

```
src/
├── app/
│   ├── (auth)/
│   │   ├── login/
│   │   ├── register/
│   │   └── layout.tsx
│   ├── (dashboard)/
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── api/
│   │   ├── auth/
│   │   ├── projects/
│   │   ├── documents/
│   │   └── chat/
│   ├── projects/
│   │   ├── [id]/
│   │   └── new/
│   ├── actions/
│   ├── error.tsx
│   ├── layout.tsx
│   ├── not-found.tsx
│   └── page.tsx
├── components/
│   ├── ui/
│   ├── common/
│   ├── auth/
│   ├── projects/
│   ├── documents/
│   ├── chat/
│   └── providers.tsx
├── hooks/
├── lib/
│   ├── auth.ts
│   ├── api-client.ts
│   └── data-fetching/
├── types/
└── styles/
```

### 11.2 パフォーマンス最適化

1. **コンポーネントの構成**:
   - 静的な部分とインタラクティブな部分を分離し、必要な箇所のみクライアントコンポーネントとして実装
   - データフェッチングはできるだけサーバーコンポーネントで行い、初期ロード時のJavaScriptペイロードを削減

2. **静的生成とISR**:
   ```tsx
   // app/blog/[slug]/page.tsx
   export async function generateStaticParams() {
     // 静的に生成するパスを指定
     const posts = await fetchAllPostSlugs();
     return posts.map((post) => ({
       slug: post.slug,
     }));
   }
   
   // ISR (Incremental Static Regeneration) の設定
   export const revalidate = 3600; // 1時間ごとに再検証
   ```

3. **画像の最適化**:
   ```tsx
   import Image from 'next/image';
   
   // 最適化された画像表示
   <Image
     src={document.thumbnailUrl}
     alt={document.title}
     width={300}
     height={400}
     priority={isHeroImage} // 優先的に読み込み
     quality={80}
     placeholder="blur"
     blurDataURL={document.blurDataUrl}
   />
   ```

### 11.3 SEO対策

```tsx
// app/projects/[id]/page.tsx
import { Metadata } from 'next';
import { fetchProject } from '@/lib/data-fetching/projects';

// 動的メタデータの生成
export async function generateMetadata({
  params
}: {
  params: { id: string }
}): Promise<Metadata> {
  const project = await fetchProject(params.id);
  
  if (!project) {
    return {
      title: 'プロジェクトが見つかりません',
    };
  }
  
  return {
    title: `${project.name} | LLM学術文書支援システム`,
    description: project.description || '学術文書の理解と学習を支援するプラットフォーム',
    openGraph: {
      title: `${project.name} | LLM学術文書支援システム`,
      description: project.description || '学術文書の理解と学習を支援するプラットフォーム',
      type: 'website',
      url: `https://llm-academic-assist.example.com/projects/${params.id}`,
    },
  };
}
```

## 12. まとめ

Next.js App Routerを使用した実装では、次のような利点があります：

1. **サーバーコンポーネントの活用**:
   - 初期ロード時のJavaScriptバンドルサイズの削減
   - APIキーなどの機密情報をクライアントに露出せず処理可能
   - データベースへの直接アクセスをサーバーコンポーネントから実現

2. **ストリーミングレスポンスの実装**:
   - LLMのストリーミング出力をリアルタイムで表示
   - Suspenseによる段階的UIレンダリング
   - ユーザー体験の向上

3. **開発効率の向上**:
   - ファイルベースの直感的なルーティング
   - エラーハンドリングとローディング状態の宣言的実装
   - Server Actionsによるフォーム処理の簡素化

4. **学術文書特有の機能実装**:
   - 数式レンダリング
   - PDF文書表示と操作
   - ストリーミングベースのチャットインターフェース

Next.js App Routerは、LLM学術文書支援システムのようなデータ駆動型アプリケーションに特に適しており、高いパフォーマンスとユーザー体験を実現するための最適な選択肢と言えます。

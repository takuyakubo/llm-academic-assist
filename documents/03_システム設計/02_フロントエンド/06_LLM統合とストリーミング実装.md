# LLM統合とストリーミング実装

## 概要

Next.js 14のApp Routerにおいて、LLM (大規模言語モデル) APIとの統合方法とストリーミングレスポンスの実装について説明します。学術文書支援システムでは、Anthropic ClaudeとOpenAI GPT-4を主に活用します。

## LLM連携の基本アーキテクチャ

```
[Frontend] <-> [Server Actions] <-> [LLM サービスクライアント] <-> [Claude/GPT-4 API]
```

## LLMサービスクライアントの実装

```typescript
// lib/llm/client.ts
import { OpenAI } from 'openai';
import { Anthropic } from '@anthropic-ai/sdk';
import { env } from '@/env.mjs';

// OpenAI クライアント
export const openai = new OpenAI({
  apiKey: env.OPENAI_API_KEY,
});

// Anthropic クライアント
export const anthropic = new Anthropic({
  apiKey: env.ANTHROPIC_API_KEY,
});

// モデル設定
export const MODELS = {
  GPT4: 'gpt-4-0125-preview',
  GPT4VISION: 'gpt-4-vision-preview',
  CLAUDE3OPUS: 'claude-3-opus-20240229',
  CLAUDE3SONNET: 'claude-3-sonnet-20240229',
  CLAUDE3HAIKU: 'claude-3-haiku-20240307',
};

// モデル選択ロジック
export function selectModel(task: 'chat' | 'summarize' | 'extract' | 'analyze'): string {
  switch (task) {
    case 'chat':
      return MODELS.CLAUDE3OPUS;
    case 'summarize':
      return MODELS.CLAUDE3SONNET;
    case 'extract':
      return MODELS.GPT4VISION;
    case 'analyze':
      return MODELS.GPT4;
    default:
      return MODELS.CLAUDE3SONNET;
  }
}
```

## ストリーミングレスポンスの実装

Next.js 14のApp Routerでは、React Suspenseと組み合わせてストリーミングレスポンスを実装できます。

```typescript
// app/api/chat/route.ts
import { NextRequest } from 'next/server';
import { anthropic } from '@/lib/llm/client';
import { auth } from '@/lib/auth';

export async function POST(req: NextRequest) {
  const { documentId, message } = await req.json();
  const userId = await auth.getUserId();
  
  if (!userId) {
    return Response.json({ error: '認証が必要です' }, { status: 401 });
  }
  
  // ドキュメントコンテキストの取得
  const document = await getDocumentContent(documentId, userId);
  
  // プロンプト構築
  const systemPrompt = `
    あなたは学術文書のアシスタントです。以下の文書内容に基づいて質問に回答してください。
    回答には文書内の情報のみを使用し、出典を明示してください。
    
    文書内容:
    ${document.content}
  `;
  
  // ストリーミングレスポンスのためのエンコーダー
  const encoder = new TextEncoder();
  
  // ストリームを設定
  const stream = new ReadableStream({
    async start(controller) {
      const messageAccumulator = [];
      
      try {
        const stream = await anthropic.messages.stream({
          model: MODELS.CLAUDE3OPUS,
          system: systemPrompt,
          messages: [
            { role: 'user', content: message }
          ],
          max_tokens: 2000,
        });
        
        for await (const chunk of stream) {
          if (chunk.type === 'content_block_delta' && chunk.delta.text) {
            // クライアントにテキストチャンクを送信
            controller.enqueue(encoder.encode(chunk.delta.text));
            
            // メッセージを蓄積（後でDBに保存するため）
            messageAccumulator.push(chunk.delta.text);
          }
        }
        
        // DBにチャット履歴を保存
        await saveMessageHistory(userId, documentId, message, messageAccumulator.join(''));
        
        controller.close();
      } catch (error) {
        controller.error(error);
      }
    }
  });
  
  // ストリーミングレスポンスを返す
  return new Response(stream);
}
```

## クライアントサイドでのストリームの処理

クライアントサイドでストリーミングレスポンスを処理するコンポーネント：

```tsx
// app/(dashboard)/projects/[id]/documents/[docId]/chat/chat-component.tsx
'use client';

import { useEffect, useState } from 'react';
import { useChat } from '@/hooks/use-chat';
import { Message } from '@/types/chat';
import MarkdownRenderer from '@/components/markdown-renderer';

export function ChatComponent({ documentId }: { documentId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  
  const sendMessage = async () => {
    if (!input.trim()) return;
    
    const userMessage: Message = {
      id: Date.now().toString(),
      role: 'user',
      content: input,
      createdAt: new Date(),
    };
    
    setMessages((prev) => [...prev, userMessage]);
    setInput('');
    setIsLoading(true);
    
    // プレースホルダーのアシスタントメッセージを追加
    const assistantMessage: Message = {
      id: (Date.now() + 1).toString(),
      role: 'assistant',
      content: '',
      createdAt: new Date(),
    };
    
    setMessages((prev) => [...prev, assistantMessage]);
    
    try {
      const response = await fetch('/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ documentId, message: input }),
      });
      
      if (!response.ok) throw new Error('エラーが発生しました');
      
      const reader = response.body?.getReader();
      const decoder = new TextDecoder();
      
      if (!reader) throw new Error('レスポンスストリームを読み込めません');
      
      let responseText = '';
      
      while (true) {
        const { done, value } = await reader.read();
        
        if (done) break;
        
        const chunk = decoder.decode(value, { stream: true });
        responseText += chunk;
        
        // 最新のメッセージを更新
        setMessages((prev) => 
          prev.map((msg, i) => 
            i === prev.length - 1 ? { ...msg, content: responseText } : msg
          )
        );
      }
    } catch (error) {
      console.error('Error in chat stream:', error);
      
      // エラーメッセージを表示
      setMessages((prev) => 
        prev.map((msg, i) => 
          i === prev.length - 1 ? { ...msg, content: 'エラーが発生しました。もう一度お試しください。' } : msg
        )
      );
    } finally {
      setIsLoading(false);
    }
  };
  
  return (
    <div className="chat-container">
      <div className="messages-container">
        {messages.map((message) => (
          <div key={message.id} className={`message ${message.role}`}>
            <div className="message-avatar">
              {message.role === 'user' ? '👤' : '🤖'}
            </div>
            <div className="message-content">
              <MarkdownRenderer content={message.content} />
            </div>
          </div>
        ))}
      </div>
      
      <div className="input-container">
        <textarea
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="質問を入力してください..."
          disabled={isLoading}
        />
        <button 
          onClick={sendMessage} 
          disabled={isLoading || !input.trim()}
        >
          {isLoading ? '応答中...' : '送信'}
        </button>
      </div>
    </div>
  );
}
```

## プロンプトエンジニアリング

学術文書支援に特化したプロンプトテンプレートを定義します：

```typescript
// lib/llm/prompts.ts
export const SYSTEM_PROMPTS = {
  DOCUMENT_SUMMARIZATION: `
    あなたは学術文書の要約エキスパートです。提供される学術文書のセクションに基づいて、
    簡潔かつ正確な要約を作成してください。特に以下の点に注意してください：
    
    1. 主要な概念、理論、手法を優先的に抽出する
    2. 数式や専門用語をそのまま保持し、簡略化しすぎない
    3. 原文の構造と論理的な流れを反映させる
    4. 冗長な表現を避け、情報密度の高い要約を作成する
    5. 著者の主張や結論を明確に示す
    
    要約には、そのセクションで登場する重要な数式や定義も含めてください。
    ただし、要約全体が{maxLength}文字程度に収まるようにしてください。
  `,
  
  CONCEPT_EXPLANATION: `
    あなたは専門的な学術概念を説明するチューターです。提供される学術文書と質問に基づいて、
    概念の詳細な説明を提供してください。特に以下の点に注意してください：
    
    1. 概念の定義と背景を明確に説明する
    2. 必要に応じて数式を使用して説明を補強する
    3. 可能であれば直感的な例や類推を提供する
    4. 関連する概念との関係性を示す
    5. 段階的に複雑さを増していく説明を心がける
    
    回答には文書から得られる情報のみを使用し、不確かな情報は推測せず「文書内にその情報はありません」と明示してください。
  `,
  
  DOCUMENT_STRUCTURE_EXTRACTION: `
    あなたは学術文書の構造を解析するエキスパートです。提供されるPDFから抽出されたテキストに基づいて、
    文書の階層構造（章・節・小節）を抽出してください。結果はJSON形式で返してください。
    
    以下の点に注意してください：
    1. 章・節・小節の番号と見出しテキストを正確に抽出する
    2. 階層関係を正しく表現する
    3. 各セクションの開始ページ番号を含める
    4. 明らかに見出しではないテキストを除外する
    
    出力形式：
    {
      "structure": [
        {
          "level": 1,
          "number": "1",
          "title": "章タイトル",
          "page": 1,
          "subsections": [
            {
              "level": 2,
              "number": "1.1",
              "title": "節タイトル",
              "page": 2,
              "subsections": []
            }
          ]
        }
      ]
    }
  `,
};
```

## 数式とLaTeXの処理

学術文書の数式をLaTeX形式で処理するためのユーティリティ関数：

```typescript
// lib/latex-utils.ts
import katex from 'katex';

// インラインの数式をレンダリング
export function renderInlineMath(tex: string): string {
  try {
    return katex.renderToString(tex, { displayMode: false });
  } catch (error) {
    console.error('KaTeX error:', error);
    return `<span class="error">${tex}</span>`;
  }
}

// ディスプレイモードの数式をレンダリング
export function renderDisplayMath(tex: string): string {
  try {
    return katex.renderToString(tex, { displayMode: true });
  } catch (error) {
    console.error('KaTeX error:', error);
    return `<div class="error">${tex}</div>`;
  }
}

// LLMレスポンス内の数式を処理
export function processLatexInMarkdown(text: string): string {
  // インライン数式（$...$）を処理
  text = text.replace(/\$([^$]+)\$/g, (_, tex) => {
    return renderInlineMath(tex);
  });
  
  // ディスプレイモード数式（$$...$$）を処理
  text = text.replace(/\$\$([^$]+)\$\$/g, (_, tex) => {
    return renderDisplayMath(tex);
  });
  
  return text;
}
```

## まとめ

LLM統合とストリーミング実装の主なポイント：

1. サーバーサイドでLLMへのリクエストを管理し、ストリーミングレスポンスを設定
2. クライアントサイドでストリームを受信し、UIにリアルタイムで表示
3. 学術文書特有のプロンプトテンプレートを設計
4. 数式とLaTeXの適切な処理と表示
5. レスポンスの保存とコンテキスト管理

これらの実装により、ユーザーは学術文書に関する質問をリアルタイムで処理でき、数式や図表を含む高度な学術内容も適切に表示されます。

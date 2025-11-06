# エンタープライズ Agentic RAG システム - 調査結果と実装計画

## 目次
1. [システム概要](#1-システム概要)
2. [Vercel AI SDK の主要機能分析](#2-vercel-ai-sdk-の主要機能分析)
3. [エンタープライズレベルの要件定義](#3-エンタープライズレベルの要件定義)
4. [アーキテクチャ設計](#4-アーキテクチャ設計)
5. [実装計画とロードマップ](#5-実装計画とロードマップ)
6. [技術スタック推奨](#6-技術スタック推奨)
7. [コード例とファイル構造](#7-コード例とファイル構造)
8. [成功のためのベストプラクティス](#8-成功のためのベストプラクティス)
9. [リスクと対策](#9-リスクと対策)
10. [次のステップ](#10-次のステップ)

---

## 1. システム概要

### Agentic RAG とは
従来の RAG（Retrieval-Augmented Generation）を進化させ、以下の特徴を持つシステム：
- **自律的な意思決定**: エージェントが複数のデータソースから動的に情報を取得
- **マルチステップ推論**: 複雑なタスクを分解して段階的に実行
- **ツール呼び出し**: 外部 API、データベース、検索エンジンなどを動的に利用
- **計画と実行の統合**: 検索と生成が動的にインターリーブされる

---

## 2. Vercel AI SDK の主要機能分析

### 利用可能な機能

#### 2.1 エージェントフレームワーク
- **ToolLoopAgent**: `/packages/ai/src/agent/tool-loop-agent.ts`
  - マルチステップ実行ループ
  - 最大 20 ステップ（デフォルト、カスタマイズ可能）
  - `onStepFinish` / `onFinish` コールバック
  - 型安全な UI メッセージ推論

#### 2.2 ツール実行システム
- **静的ツール**: コンパイル時の型チェック
- **動的ツール**: ランタイム発見（MCP サポート）
- **Human-in-the-Loop**: `execute` 関数なしで承認フロー
- **ストリーミングツール結果**: 進行中の結果をリアルタイム表示

#### 2.3 RAG 統合パターン
- **Middleware アーキテクチャ**: `LanguageModelV3Middleware`
  - プロンプト変換
  - コンテキスト注入
  - チェーン可能な設計
- **埋め込み生成**: `embed()` / `embedMany()`
  - OpenAI, Google, Cohere, Mistral などをサポート

#### 2.4 エンタープライズ機能
- **再試行ロジック**: 指数バックオフ（デフォルト 2 回）
- **OpenTelemetry 統合**: 自動スパン作成
  - `ai.toolCall`, `ai.generateText`, `ai.streamText`
  - 入力/出力の記録オプション
- **プロバイダーレジストリ**: 40+ のモデルプロバイダー
- **ストリーミング**: `AsyncIterableStream` でブラウザ/Node.js 両対応

### 主要ファイルの場所

**Core Generation**:
- `/packages/ai/src/generate-text/generate-text.ts` (非ストリーミング)
- `/packages/ai/src/generate-text/stream-text.ts` (ストリーミング)

**Agent & Tools**:
- `/packages/ai/src/agent/tool-loop-agent.ts`
- `/packages/ai/src/generate-text/execute-tool-call.ts`

**Streaming & State**:
- `/packages/ai/src/util/async-iterable-stream.ts`
- `/packages/ai/src/ui/chat.ts`

**Enterprise Features**:
- `/packages/ai/src/telemetry/` (可観測性)
- `/packages/ai/src/util/retry-with-exponential-backoff.ts` (耐障害性)
- `/packages/ai/src/middleware/wrap-language-model.ts` (横断的関心事)

**RAG Examples**:
- `/examples/ai-core/src/middleware/your-rag-middleware.ts`
- `/examples/ai-core/src/middleware/stream-text-rag-middleware.ts`

---

## 3. エンタープライズレベルの要件定義

### 3.1 機能要件
- ✅ **マルチソース検索**: ベクトル DB、SQL、API から横断検索
- ✅ **階層的検索戦略**: Naive → Advanced RAG → Agentic RAG
- ✅ **検証エージェント**: 取得した情報の正確性を検証
- ✅ **メモリ管理**: 会話履歴とコンテキストの永続化
- ✅ **動的ルーティング**: クエリタイプに応じた適切なツール選択

### 3.2 非機能要件
- ✅ **スケーラビリティ**: 同時実行数 1000+ リクエスト
- ✅ **レイテンシ**: P95 < 3秒（初回トークン生成）
- ✅ **可観測性**:
  - リアルタイムモニタリング
  - 分散トレーシング
  - メトリクス収集（Precision, Recall, F1, NDCG）
- ✅ **セキュリティ**:
  - アクセス制御（RBAC）
  - データ暗号化（転送時/保管時）
  - 監査ログ
- ✅ **信頼性**:
  - サーキットブレーカー
  - レート制限
  - フォールバック戦略

### 3.3 評価要件
- ✅ **オフライン評価**: 開発時の品質検証
- ✅ **オンライン評価**: 本番環境でのA/Bテスト
- ✅ **継続的評価**: CI/CD パイプラインとの統合

---

## 4. アーキテクチャ設計

### 4.1 システムアーキテクチャ

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend (React/Next.js)                  │
│                      useChat Hook                            │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                  API Layer (Next.js API Routes)              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │           Agentic RAG Orchestrator                     │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │         ToolLoopAgent (Core Agent)               │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────┬──────────────┬──────────────┬────────────────┘
               │              │              │
      ┌────────▼─────┐  ┌────▼─────┐  ┌────▼─────────┐
      │ Query        │  │ Retrieval│  │ Validation   │
      │ Planning     │  │ Agent    │  │ Agent        │
      │ Tool         │  │ Tool     │  │ Tool         │
      └────┬─────────┘  └────┬─────┘  └────┬─────────┘
           │                 │              │
┌──────────▼─────────────────▼──────────────▼─────────────────┐
│                    RAG Middleware Layer                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Query       │  │ Context     │  │ Response    │         │
│  │ Rewriting   │  │ Enrichment  │  │ Synthesis   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└──────────────────────┬───────────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────────┐
│                 Data Source Layer                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Vector   │  │ Knowledge│  │ SQL      │  │ External │   │
│  │ Store    │  │ Graph    │  │ Database │  │ APIs     │   │
│  │ (Pinecone│  │ (Neo4j)  │  │ (Postgres│  │          │   │
│  │ /Qdrant) │  │          │  │ /MySQL)  │  │          │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│            Observability & Monitoring Layer                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │OpenTelem │  │ Metrics  │  │ Tracing  │  │ Logging  │   │
│  │  etry    │  │(Prometheus│ │(Jaeger/  │  │(Winston/ │   │
│  │          │  │ /Grafana)│  │DataDog)  │  │Pino)     │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 コンポーネント詳細

#### A. Agentic RAG Orchestrator
```typescript
// 基本構造
import { ToolLoopAgent } from 'ai';
import { openai } from '@ai-sdk/openai';
import { wrapLanguageModel } from 'ai';

export const agenticRagOrchestrator = new ToolLoopAgent({
  model: wrapLanguageModel({
    model: openai('gpt-4o'),
    middleware: ragMiddleware, // RAG 機能を注入
  }),
  instructions: `あなたはエンタープライズ向け情報検索エージェントです。
  複数のソースから情報を収集し、検証し、正確な回答を提供してください。`,
  tools: {
    queryPlanning: queryPlanningTool,
    vectorSearch: vectorSearchTool,
    sqlQuery: sqlQueryTool,
    apiCall: apiCallTool,
    validation: validationTool,
  },
  onStepFinish: async ({ step }) => {
    // OpenTelemetry でトレーシング
    await logStepToObservability(step);
  },
});
```

#### B. RAG Middleware
```typescript
import { LanguageModelV3Middleware } from '@ai-sdk/provider';
import { embed } from 'ai';

export const ragMiddleware: LanguageModelV3Middleware = {
  specificationVersion: 'v3',
  transformParams: async ({ params }) => {
    const query = getLastUserMessageText({ prompt: params.prompt });

    if (!query) return params;

    // 1. クエリ分析と埋め込み生成
    const { embedding } = await embed({
      model: registry.textEmbeddingModel('openai:text-embedding-3-large'),
      value: query,
    });

    // 2. ベクトル検索
    const retrievedDocs = await vectorStore.search(embedding, { topK: 10 });

    // 3. リランキング
    const rerankedDocs = await rerank(query, retrievedDocs);

    // 4. コンテキスト注入
    const context = formatContext(rerankedDocs);
    return addToLastUserMessage({
      params,
      text: `Context:\n${context}\n\nQuestion: ${query}`
    });
  },
};
```

#### C. ツール定義例

**Vector Search Tool**
```typescript
import { tool } from 'ai';
import { z } from 'zod';

export const vectorSearchTool = tool({
  description: 'ベクトルデータベースから関連文書を検索',
  parameters: z.object({
    query: z.string().describe('検索クエリ'),
    topK: z.number().default(10).describe('取得する文書数'),
    filters: z.record(z.any()).optional().describe('フィルタ条件'),
  }),
  execute: async ({ query, topK, filters }) => {
    const { embedding } = await embed({
      model: registry.textEmbeddingModel('openai:text-embedding-3-large'),
      value: query,
    });

    const results = await vectorStore.query({
      vector: embedding,
      topK,
      filter: filters,
    });

    return {
      documents: results.matches.map(m => ({
        content: m.metadata.text,
        score: m.score,
        source: m.metadata.source,
      })),
    };
  },
});
```

**Validation Tool**
```typescript
export const validationTool = tool({
  description: '取得した情報の正確性を検証',
  parameters: z.object({
    claim: z.string().describe('検証する主張'),
    sources: z.array(z.string()).describe('ソース文書'),
  }),
  execute: async ({ claim, sources }) => {
    // Chain-of-Thought で検証
    const result = await generateText({
      model: openai('gpt-4o'),
      prompt: `以下の主張がソース文書によって裏付けられるか検証してください：

主張: ${claim}

ソース:
${sources.join('\n\n')}

検証結果をJSON形式で返してください：
{
  "verified": boolean,
  "confidence": number (0-1),
  "reasoning": string
}`,
    });

    return JSON.parse(result.text);
  },
});
```

**Query Planning Tool**
```typescript
export const queryPlanningTool = tool({
  description: '複雑なクエリを分解して実行計画を作成',
  parameters: z.object({
    query: z.string().describe('ユーザーのクエリ'),
  }),
  execute: async ({ query }) => {
    const result = await generateText({
      model: openai('gpt-4o'),
      prompt: `以下のクエリを分析し、実行計画を作成してください：

クエリ: ${query}

実行計画をJSON形式で返してください：
{
  "steps": [
    {
      "action": "vector_search" | "sql_query" | "api_call",
      "description": string,
      "parameters": object
    }
  ]
}`,
    });

    return JSON.parse(result.text);
  },
});
```

---

## 5. 実装計画とロードマップ

### Phase 1: 基盤構築（2-3週間）

#### 1.1 プロジェクトセットアップ
- [ ] Next.js + TypeScript プロジェクト作成
- [ ] Vercel AI SDK インストール (`npm install ai @ai-sdk/openai @ai-sdk/anthropic`)
- [ ] 環境変数設定（API キー等）
- [ ] ディレクトリ構造設計

#### 1.2 ベクトルストア統合
- [ ] ベクトルDB選定（Pinecone / Qdrant / Weaviate）
- [ ] 埋め込みモデル選定（OpenAI text-embedding-3-large 推奨）
- [ ] データインジェストパイプライン構築
  - [ ] チャンキング戦略実装（512-1024トークン）
  - [ ] メタデータ抽出
  - [ ] 埋め込み生成とアップロード

#### 1.3 基本 RAG 実装
- [ ] RAG Middleware 実装
  - [ ] クエリ埋め込み生成
  - [ ] ベクトル検索
  - [ ] コンテキスト注入
- [ ] 基本的な `streamText` API 実装
- [ ] React `useChat` フック統合

**成果物**: 基本的な RAG チャットアプリケーション

---

### Phase 2: Agentic 機能追加（3-4週間）

#### 2.1 ToolLoopAgent セットアップ
- [ ] Agent の基本構造実装
- [ ] ツールセット定義
- [ ] ステップ制御ロジック（`maxSteps`, 停止条件）

#### 2.2 コアツール実装
- [ ] **Query Planning Tool**: クエリ分解と実行計画
- [ ] **Vector Search Tool**: 高度な検索とフィルタリング
- [ ] **SQL Query Tool**: 構造化データ検索
- [ ] **API Call Tool**: 外部 API 統合
- [ ] **Validation Tool**: 情報検証

#### 2.3 Multi-Agent 協調
- [ ] Retrieval Agent（情報収集特化）
- [ ] Validation Agent（検証特化）
- [ ] Synthesis Agent（統合特化）
- [ ] Agent 間通信プロトコル

#### 2.4 メモリ管理
- [ ] 会話履歴の永続化（PostgreSQL or KV store）
- [ ] コンテキストウィンドウ管理
- [ ] セッション管理

**成果物**: 自律的に情報を収集・検証できる Agentic RAG システム

---

### Phase 3: エンタープライズ機能（3-4週間）

#### 3.1 可観測性
- [ ] OpenTelemetry セットアップ
  - [ ] カスタムスパン追加
  - [ ] メトリクス収集
- [ ] ログ集約（Winston / Pino）
- [ ] ダッシュボード構築（Grafana / DataDog）

#### 3.2 セキュリティ
- [ ] 認証・認可（NextAuth.js / Auth0）
- [ ] RBAC 実装
- [ ] レート制限（Upstash Redis）
- [ ] 入力バリデーション（Zod）
- [ ] 監査ログ

#### 3.3 信頼性向上
- [ ] サーキットブレーカー実装
- [ ] フォールバック戦略
- [ ] エラーハンドリング強化
- [ ] リトライロジックのカスタマイズ

#### 3.4 パフォーマンス最適化
- [ ] キャッシング戦略（Prompt / Response / Embedding）
- [ ] バッチ処理
- [ ] ストリーミング最適化
- [ ] 並列処理

**成果物**: 本番運用可能なエンタープライズシステム

---

### Phase 4: 評価と改善（継続的）

#### 4.1 評価フレームワーク
- [ ] オフライン評価
  - [ ] テストデータセット作成
  - [ ] メトリクス実装（Precision, Recall, F1, NDCG）
  - [ ] 自動テストスイート
- [ ] オンライン評価
  - [ ] A/B テストフレームワーク
  - [ ] ユーザーフィードバック収集
  - [ ] リアルタイムメトリクス

#### 4.2 CI/CD 統合
- [ ] 自動評価をパイプラインに統合
- [ ] パフォーマンスリグレッション検出
- [ ] 段階的デプロイ（Canary / Blue-Green）

#### 4.3 継続的改善
- [ ] ユーザーフィードバックループ
- [ ] モデルファインチューニング
- [ ] プロンプトエンジニアリング最適化
- [ ] 検索戦略の A/B テスト

**成果物**: 自己改善可能な RAG システム

---

## 6. 技術スタック推奨

### フロントエンド
- **Next.js 15**: React フレームワーク
- **Vercel AI SDK**: `ai` パッケージ（useChat, useAgent）
- **TailwindCSS**: スタイリング
- **shadcn/ui**: UI コンポーネント

### バックエンド
- **Next.js API Routes**: サーバーサイドロジック
- **Vercel AI SDK**:
  - `ai`: コア機能
  - `@ai-sdk/openai`: OpenAI 統合
  - `@ai-sdk/anthropic`: Anthropic 統合
- **Zod**: スキーマバリデーション

### データストレージ
- **Pinecone / Qdrant / Weaviate**: ベクトルデータベース
- **PostgreSQL**: メタデータと会話履歴
- **Redis (Upstash)**: キャッシングとレート制限

### 可観測性
- **OpenTelemetry**: 分散トレーシング
- **Prometheus**: メトリクス収集
- **Grafana**: 可視化
- **DataDog / New Relic**: APM（オプション）

### 評価ツール
- **RAGAS**: RAG 評価メトリクス
- **LangSmith**: LLM アプリ監視
- **Braintrust**: 評価とモニタリング

### インフラ
- **Vercel**: ホスティング
- **Docker**: コンテナ化
- **Kubernetes**: オーケストレーション（大規模の場合）

---

## 7. コード例とファイル構造

### 推奨ディレクトリ構造
```
enterprise-agentic-rag/
├── app/
│   ├── api/
│   │   ├── chat/
│   │   │   └── route.ts          # メイン Chat API
│   │   └── agent/
│   │       └── route.ts          # Agent API
│   ├── page.tsx                  # チャット UI
│   └── layout.tsx
├── lib/
│   ├── agent/
│   │   ├── orchestrator.ts       # メインエージェント
│   │   ├── tools/
│   │   │   ├── vector-search.ts
│   │   │   ├── sql-query.ts
│   │   │   ├── validation.ts
│   │   │   ├── query-planning.ts
│   │   │   └── index.ts
│   │   └── middleware/
│   │       └── rag-middleware.ts
│   ├── db/
│   │   ├── vector-store.ts       # ベクトルDB クライアント
│   │   ├── postgres.ts           # SQL クライアント
│   │   └── redis.ts              # キャッシュクライアント
│   ├── observability/
│   │   ├── telemetry.ts          # OpenTelemetry 設定
│   │   ├── logger.ts             # ロガー設定
│   │   └── metrics.ts            # カスタムメトリクス
│   └── utils/
│       ├── embeddings.ts
│       ├── chunking.ts
│       ├── reranking.ts
│       └── validation.ts
├── evaluation/
│   ├── datasets/                 # テストデータ
│   │   ├── test-queries.json
│   │   └── ground-truth.json
│   ├── metrics/                  # 評価メトリクス
│   │   ├── precision.ts
│   │   ├── recall.ts
│   │   └── ndcg.ts
│   └── run-eval.ts              # 評価スクリプト
├── scripts/
│   ├── ingest-data.ts           # データインジェスト
│   └── migrate.ts               # DB マイグレーション
├── components/
│   ├── chat-interface.tsx
│   ├── message-list.tsx
│   └── source-citation.tsx
├── .env.local
├── tsconfig.json
├── package.json
└── README.md
```

### Chat API 実装例 (`app/api/chat/route.ts`)

```typescript
import { streamText } from 'ai';
import { agenticRagOrchestrator } from '@/lib/agent/orchestrator';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = agenticRagOrchestrator.streamUI({
    messages,
  });

  return result.toDataStreamResponse();
}
```

### フロントエンド実装例 (`app/page.tsx`)

```typescript
'use client';

import { useChat } from 'ai/react';

export default function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } =
    useChat({
      api: '/api/chat',
    });

  return (
    <div className="flex flex-col h-screen">
      <div className="flex-1 overflow-y-auto">
        {messages.map(m => (
          <div key={m.id} className={`message ${m.role}`}>
            <strong>{m.role}:</strong>
            <div>{m.content}</div>

            {/* ツール呼び出しの表示 */}
            {m.toolInvocations?.map(tool => (
              <div key={tool.toolCallId} className="tool-call">
                <span>🔧 {tool.toolName}</span>
                {tool.result && (
                  <pre>{JSON.stringify(tool.result, null, 2)}</pre>
                )}
              </div>
            ))}
          </div>
        ))}
      </div>

      <form onSubmit={handleSubmit} className="p-4">
        <input
          value={input}
          onChange={handleInputChange}
          disabled={isLoading}
          placeholder="質問を入力..."
          className="w-full p-2 border rounded"
        />
      </form>
    </div>
  );
}
```

---

## 8. 成功のためのベストプラクティス

### 8.1 データ品質
- ✅ **高品質なチャンキング**: 512-1024トークン推奨、文脈を保持
- ✅ **リッチなメタデータ**: 日付、著者、カテゴリ、ソースURL等
- ✅ **定期的なデータ更新**: 増分更新パイプライン構築
- ✅ **データクリーニング**: 重複除去、ノイズ除去

### 8.2 プロンプトエンジニアリング
- ✅ **構造化プロンプト**: System / User / Assistant を明確に分離
- ✅ **Few-shot Examples**: 期待する出力形式の例を提供
- ✅ **Chain-of-Thought**: ステップバイステップの推論を促す
- ✅ **制約の明示**: 「ソースに基づいてのみ回答」など

### 8.3 検索戦略
- ✅ **ハイブリッド検索**: ベクトル検索 + キーワード検索
- ✅ **リランキング**: Cohere Rerank などで精度向上
- ✅ **フィルタリング**: メタデータベースの事前フィルタ
- ✅ **クエリ拡張**: 同義語展開、クエリリライト

### 8.4 コスト最適化
- ✅ **プロンプトキャッシング**: Anthropic / OpenAI のキャッシュ機能活用
- ✅ **適切なモデル選択**:
  - 複雑なタスク: GPT-4o, Claude 3.5 Sonnet
  - シンプルなタスク: GPT-4o-mini, Claude 3 Haiku
- ✅ **埋め込みモデルの効率化**: text-embedding-3-small で十分な場合も
- ✅ **レスポンスキャッシング**: 同じクエリは再利用

### 8.5 ユーザーエクスペリエンス
- ✅ **ストリーミングレスポンス**: 体感速度の向上
- ✅ **ソース引用の表示**: 透明性と信頼性
- ✅ **検証ステータスの可視化**: エージェントの思考プロセス表示
- ✅ **Human-in-the-Loop**: 重要な操作には承認フロー

### 8.6 評価とモニタリング
- ✅ **継続的評価**: 本番データで定期的に評価
- ✅ **メトリクス追跡**:
  - 検索精度: Precision@K, Recall@K, NDCG
  - 生成品質: BLEU, ROUGE, BERTScore
  - ユーザー満足度: Thumbs up/down, NPS
- ✅ **A/B テスト**: 新しい戦略の効果測定
- ✅ **異常検知**: レイテンシ、エラー率の監視

---

## 9. リスクと対策

| リスク | 影響度 | 発生確率 | 対策 |
|--------|--------|----------|------|
| **LLM のハルシネーション** | 高 | 中 | • Validation Agent によるファクトチェック<br>• ソース引用の義務化<br>• 信頼度スコアの表示 |
| **レイテンシ増加** | 中 | 高 | • プロンプト/レスポンスのキャッシング<br>• 並列ツール実行<br>• ストリーミングレスポンス |
| **コスト超過** | 中 | 中 | • バジェットアラート設定<br>• レート制限<br>• 適切なモデル選択（mini モデル活用） |
| **セキュリティ侵害** | 高 | 低 | • RBAC によるアクセス制御<br>• 入力サニタイゼーション<br>• 監査ログ<br>• データ暗号化 |
| **スケーラビリティ問題** | 中 | 中 | • オートスケーリング<br>• 負荷分散<br>• キューイングシステム |
| **データ品質低下** | 高 | 中 | • データバリデーションパイプライン<br>• 定期的な品質監査<br>• ユーザーフィードバック収集 |
| **ツールの信頼性問題** | 中 | 中 | • サーキットブレーカー<br>• フォールバック戦略<br>• タイムアウト設定<br>• エラーハンドリング |
| **コンテキスト制限** | 中 | 高 | • 効率的なチャンキング<br>• コンテキスト圧縮<br>• 階層的検索（要約 → 詳細） |

---

## 10. 次のステップ

### 即座に開始できること
1. **プロジェクト初期化**
   ```bash
   npx create-next-app@latest enterprise-agentic-rag --typescript --tailwind --app
   cd enterprise-agentic-rag
   npm install ai @ai-sdk/openai zod
   ```

2. **環境変数設定** (`.env.local`)
   ```env
   OPENAI_API_KEY=your_key_here
   # ベクトルDB
   PINECONE_API_KEY=your_key_here
   PINECONE_ENVIRONMENT=your_env_here
   # データベース
   DATABASE_URL=postgresql://...
   # 可観測性
   OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
   ```

3. **POC 構築**
   - Phase 1 の基盤構築から開始
   - 小規模データセット（100-1000 文書）でテスト
   - 基本的な RAG フローを検証

### 学習リソース
- **Vercel AI SDK ドキュメント**: https://sdk.vercel.ai/docs
- **ToolLoopAgent ガイド**: https://sdk.vercel.ai/docs/ai-sdk-core/agents
- **RAG 評価**: RAGAS (https://docs.ragas.io/)
- **可観測性**: OpenTelemetry for JavaScript

### コミュニティとサポート
- **Vercel AI SDK GitHub**: https://github.com/vercel/ai
- **Discord コミュニティ**: Vercel Community
- **参考実装**: `/examples/` ディレクトリ内の実装例

---

## まとめ

この計画書は、Vercel AI SDK を活用したエンタープライズレベルの Agentic RAG システムの包括的なロードマップを提供します。

**主要な特徴**:
- ✅ 型安全な TypeScript 実装
- ✅ マルチステップエージェントによる自律的な情報検索
- ✅ 複数データソースの統合
- ✅ 本番環境対応のエンタープライズ機能
- ✅ 継続的評価と改善のフレームワーク

**推奨アプローチ**:
1. Phase 1 でMVP構築（2-3週間）
2. 実データでPOC検証
3. Phase 2-3 で機能拡張（6-8週間）
4. Phase 4 で継続的改善サイクル確立

各フェーズは独立しているため、段階的に実装を進めることが可能です。

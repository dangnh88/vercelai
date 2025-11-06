# CLAUDE.md - Project Constitution & Living Specification

> **Document Status**: Living Specification (SSOT)
> **Version**: 1.0.0
> **Last Updated**: 2025-11-06
> **Development Methodology**: Specification-Driven Development (SDD)

---

## 📜 Document Purpose

This document serves as the **Single Source of Truth (SSOT)** for the Enterprise Agentic RAG project. It is a **living specification** that:

- Defines the **intent** and **vision** of the system (not just implementation details)
- Acts as an **executable contract** between human developers and AI coding agents
- Drives all downstream artifacts: code, tests, documentation, and infrastructure
- Evolves continuously through version control
- Serves as the **project constitution** for Claude Code and other AI agents

**Development Workflow**: `Specify → Plan → Tasks → Implement`

---

## 🎯 Project Vision

### Mission Statement
Build a production-ready, enterprise-grade Agentic RAG (Retrieval-Augmented Generation) system using Vercel AI SDK that autonomously retrieves, validates, and synthesizes information from multiple data sources with human-level reasoning capabilities.

### Core Values
1. **Accuracy over Speed**: Validated information is more valuable than fast but incorrect responses
2. **Transparency**: Every claim must be traceable to its source
3. **Autonomy with Oversight**: Agents act independently but seek human approval for critical decisions
4. **Scalability**: Design for 1000+ concurrent users from day one
5. **Type Safety**: Leverage TypeScript for compile-time guarantees

### Success Criteria
- ✅ P95 latency < 3 seconds for first token
- ✅ Source attribution for 100% of factual claims
- ✅ Validation accuracy > 95% (measured against ground truth)
- ✅ Cost per query < $0.10
- ✅ System uptime > 99.5%

---

## 🏗️ System Architecture Specification

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Layer                              │
│  Next.js App Router + React Server Components               │
│  - Chat Interface (useChat hook)                            │
│  - Agent Dashboard (useAgent hook)                          │
│  - Source Citation UI                                       │
└──────────────────────┬──────────────────────────────────────┘
                       │ HTTPS/WebSocket
┌──────────────────────▼──────────────────────────────────────┐
│                  API Layer (Edge Runtime)                    │
│  - /api/chat/route.ts - Main chat endpoint                  │
│  - /api/agent/route.ts - Agent streaming endpoint           │
│  - /api/feedback/route.ts - User feedback collection        │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│            Agentic RAG Orchestrator (Core)                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  ToolLoopAgent                                       │  │
│  │  - Model: GPT-4o (primary), Claude 3.5 (fallback)   │  │
│  │  - Max Steps: 20                                     │  │
│  │  - Stop Condition: Custom logic                      │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  RAG Middleware                                      │  │
│  │  - Query Analysis → Embedding → Retrieval → Rerank  │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                   Tool Execution Layer                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Query    │  │ Vector   │  │ SQL      │  │Validation│  │
│  │ Planning │  │ Search   │  │ Query    │  │ Agent    │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                    Data Sources                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Pinecone │  │PostgreSQL│  │  Redis   │  │ External │  │
│  │ (Vectors)│  │(Metadata)│  │ (Cache)  │  │   APIs   │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│              Cross-Cutting Concerns                          │
│  - OpenTelemetry (Tracing, Metrics, Logs)                   │
│  - Authentication (NextAuth.js)                              │
│  - Rate Limiting (Upstash Redis)                             │
│  - Error Handling & Circuit Breakers                         │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔧 Technical Stack (SSOT)

### Core Framework
```yaml
framework: Next.js 15 (App Router)
runtime: Edge Runtime (for API routes)
language: TypeScript 5.3+
package_manager: pnpm
node_version: ">=20.0.0"
```

### AI & LLM
```yaml
ai_sdk:
  - ai@4.x (Vercel AI SDK Core)
  - "@ai-sdk/openai@1.x"
  - "@ai-sdk/anthropic@1.x"

models:
  primary: gpt-4o
  fallback: claude-3-5-sonnet-20241022
  embedding: text-embedding-3-large
  reranking: cohere-rerank-v3.5

agent_framework: ToolLoopAgent (Vercel AI SDK)
```

### Data Layer
```yaml
vector_database:
  provider: Pinecone
  index_dimension: 3072
  metric: cosine
  pods: serverless

relational_database:
  provider: PostgreSQL (Vercel Postgres)
  orm: Drizzle ORM
  schema_versioning: true

cache:
  provider: Upstash Redis
  use_cases: [rate_limiting, response_cache, session_store]
```

### Observability
```yaml
tracing: OpenTelemetry + DataDog APM
metrics: Prometheus + Grafana
logging: Pino (structured JSON logs)
error_tracking: Sentry

telemetry_config:
  record_inputs: true
  record_outputs: true
  sample_rate: 1.0 (dev), 0.1 (prod)
```

### Testing
```yaml
unit_tests: Vitest
integration_tests: Playwright
e2e_tests: Playwright
rag_evaluation: RAGAS + Custom Metrics

coverage_target: 80%
```

---

## 📐 Component Specifications

### 1. Agentic RAG Orchestrator

**Location**: `/lib/agent/orchestrator.ts`

**Specification**:
```typescript
/**
 * SPECIFICATION: Agentic RAG Orchestrator
 *
 * PURPOSE: Coordinate multi-step information retrieval and synthesis
 *
 * BEHAVIOR:
 * 1. Receive user query
 * 2. Plan retrieval strategy (via Query Planning Tool)
 * 3. Execute parallel searches across data sources
 * 4. Validate retrieved information (via Validation Tool)
 * 5. Synthesize final response with source citations
 * 6. Stream response to client
 *
 * CONSTRAINTS:
 * - Maximum 20 steps per query
 * - Timeout after 30 seconds
 * - Always provide source citations
 * - Reject queries that cannot be validated
 */

interface OrchestratorConfig {
  model: LanguageModel;
  tools: ToolSet;
  maxSteps: number; // Must be <= 20
  timeout: number; // Must be <= 30000ms
  middleware: LanguageModelV3Middleware[];
}

interface OrchestratorResponse {
  text: string;
  sources: Array<{
    url: string;
    title: string;
    confidence: number; // 0-1
  }>;
  validationStatus: 'verified' | 'unverified' | 'conflicting';
  steps: AgentStep[];
}
```

**Implementation Rules**:
- MUST use `wrapLanguageModel` to inject RAG middleware
- MUST implement fallback to Claude if OpenAI fails
- MUST log all tool calls to OpenTelemetry
- MUST NOT proceed if validation confidence < 0.7

---

### 2. RAG Middleware

**Location**: `/lib/agent/middleware/rag-middleware.ts`

**Specification**:
```typescript
/**
 * SPECIFICATION: RAG Middleware
 *
 * PURPOSE: Transparently inject retrieval context into LLM prompts
 *
 * BEHAVIOR:
 * 1. Extract last user message
 * 2. Generate query embedding (text-embedding-3-large)
 * 3. Perform hybrid search (vector + keyword)
 * 4. Rerank results (Cohere Rerank v3.5)
 * 5. Format top-K results as context
 * 6. Inject into system message
 *
 * PERFORMANCE:
 * - Embeddings must be cached (Redis, TTL: 1 hour)
 * - Search must complete in < 500ms (P95)
 * - Reranking must process in < 300ms (P95)
 *
 * QUALITY:
 * - Retrieve top-50 candidates, rerank to top-10
 * - Include metadata: source, timestamp, relevance score
 * - Filter out results with score < 0.5
 */

interface RagMiddlewareConfig {
  vectorStore: VectorStoreClient;
  embeddingModel: EmbeddingModel;
  rerankModel: RerankModel;
  topK: number; // Default: 10
  scoreThreshold: number; // Default: 0.5
}
```

**Implementation Rules**:
- MUST use semantic cache for embeddings
- MUST implement circuit breaker for vector DB
- MUST include source provenance in metadata
- MUST handle empty results gracefully

---

### 3. Tool Definitions

#### Query Planning Tool

**Location**: `/lib/agent/tools/query-planning.ts`

**Specification**:
```typescript
/**
 * SPECIFICATION: Query Planning Tool
 *
 * PURPOSE: Decompose complex queries into executable sub-tasks
 *
 * INPUT:
 * - query: string (user's natural language query)
 *
 * OUTPUT:
 * - ExecutionPlan with ordered steps
 *
 * BEHAVIOR:
 * 1. Classify query type (factual, comparative, aggregative, etc.)
 * 2. Identify required data sources
 * 3. Generate step-by-step execution plan
 * 4. Estimate complexity and time
 *
 * CONSTRAINTS:
 * - Must return valid JSON
 * - Maximum 10 steps per plan
 * - Each step must specify tool name and parameters
 */

const queryPlanningTool = tool({
  description: 'Decompose complex queries into executable sub-tasks',
  parameters: z.object({
    query: z.string().min(1).max(2000),
  }),
  execute: async ({ query }) => {
    // Implementation must follow specification
  },
});
```

#### Vector Search Tool

**Location**: `/lib/agent/tools/vector-search.ts`

**Specification**:
```typescript
/**
 * SPECIFICATION: Vector Search Tool
 *
 * PURPOSE: Search vector database for semantically similar documents
 *
 * INPUT:
 * - query: string
 * - topK: number (default: 10)
 * - filters: Record<string, any> (optional metadata filters)
 *
 * OUTPUT:
 * - Array of documents with content, score, metadata
 *
 * PERFORMANCE:
 * - Must complete in < 500ms (P95)
 * - Must handle timeouts gracefully
 *
 * QUALITY:
 * - Only return results with score > 0.5
 * - Include source URL in every result
 * - Deduplicate near-identical documents
 */

const vectorSearchTool = tool({
  description: 'Search vector database for semantically similar documents',
  parameters: z.object({
    query: z.string(),
    topK: z.number().int().min(1).max(50).default(10),
    filters: z.record(z.any()).optional(),
  }),
  execute: async ({ query, topK, filters }) => {
    // Implementation must follow specification
  },
});
```

#### Validation Tool

**Location**: `/lib/agent/tools/validation.ts`

**Specification**:
```typescript
/**
 * SPECIFICATION: Validation Tool
 *
 * PURPOSE: Verify factual claims against source documents
 *
 * INPUT:
 * - claim: string (the assertion to verify)
 * - sources: string[] (supporting documents)
 *
 * OUTPUT:
 * - verified: boolean
 * - confidence: number (0-1)
 * - reasoning: string (explanation of verdict)
 *
 * BEHAVIOR:
 * 1. Use Chain-of-Thought prompting
 * 2. Check for direct support, indirect support, or contradiction
 * 3. Calculate confidence based on evidence strength
 * 4. Provide detailed reasoning
 *
 * CONSTRAINTS:
 * - Must use GPT-4o for validation (no mini models)
 * - confidence < 0.7 = unverified
 * - Must detect and flag conflicting sources
 */

const validationTool = tool({
  description: 'Verify factual claims against source documents',
  parameters: z.object({
    claim: z.string(),
    sources: z.array(z.string()).min(1),
  }),
  execute: async ({ claim, sources }) => {
    // Implementation must follow specification
  },
});
```

---

## 🔐 Security Specifications

### Authentication & Authorization

```yaml
authentication:
  provider: NextAuth.js v5
  strategies: [email, google, github]
  session_duration: 7 days
  refresh_token: true

authorization:
  model: RBAC (Role-Based Access Control)
  roles:
    - admin: Full system access
    - power_user: Advanced features, higher rate limits
    - user: Basic chat functionality
    - readonly: View-only access

rate_limiting:
  free_tier: 10 requests/hour
  user_tier: 100 requests/hour
  power_user_tier: 1000 requests/hour
  admin_tier: unlimited
```

### Data Security

```yaml
data_encryption:
  at_rest: AES-256
  in_transit: TLS 1.3

sensitive_data_handling:
  pii_detection: Automatic redaction
  audit_logging: All data access logged
  retention_policy: 90 days for logs, 1 year for analytics

input_validation:
  max_query_length: 2000 characters
  sanitization: HTML/SQL injection prevention
  rate_limiting: Per-user and per-IP
```

---

## 📊 Data Flow Specification

### Query Processing Flow

```
1. USER INPUT
   ↓
2. AUTHENTICATION CHECK
   ↓ (if authenticated)
3. RATE LIMIT CHECK
   ↓ (if under limit)
4. INPUT VALIDATION & SANITIZATION
   ↓
5. QUERY PLANNING
   ├─→ Classify query type
   ├─→ Identify data sources
   └─→ Generate execution plan
   ↓
6. PARALLEL RETRIEVAL
   ├─→ Vector Search (Pinecone)
   ├─→ SQL Query (PostgreSQL) [if needed]
   └─→ API Calls (External) [if needed]
   ↓
7. RESULT AGGREGATION
   ↓
8. RERANKING (Cohere)
   ↓
9. VALIDATION
   ├─→ Verify each claim
   ├─→ Calculate confidence scores
   └─→ Flag conflicts
   ↓
10. RESPONSE SYNTHESIS
    ├─→ Generate natural language response
    ├─→ Inject source citations
    └─→ Add validation metadata
    ↓
11. STREAMING RESPONSE
    ↓
12. LOG TO OBSERVABILITY PLATFORM
    ↓
13. USER FEEDBACK COLLECTION [optional]
```

---

## 🧪 Testing Strategy

### Test Pyramid

```yaml
unit_tests:
  coverage_target: 80%
  framework: Vitest
  location: "**/*.test.ts"
  run_on: [pre-commit, ci/cd]

integration_tests:
  coverage_target: 70%
  framework: Vitest + MSW (API mocking)
  location: "**/*.integration.test.ts"
  run_on: [ci/cd]

e2e_tests:
  framework: Playwright
  browsers: [chromium, firefox, webkit]
  location: "e2e/**/*.spec.ts"
  run_on: [ci/cd, nightly]

rag_evaluation:
  framework: RAGAS + Custom
  metrics:
    - context_precision
    - context_recall
    - answer_relevancy
    - faithfulness
  dataset: evaluation/datasets/ground-truth.json
  run_on: [ci/cd, weekly]
```

### Test Data Management

```yaml
test_datasets:
  location: evaluation/datasets/
  format: JSON Lines
  versioning: Git LFS

datasets:
  - name: factual-queries
    size: 500 examples
    use: Basic RAG testing

  - name: complex-queries
    size: 200 examples
    use: Multi-step agent testing

  - name: adversarial-queries
    size: 100 examples
    use: Robustness testing (hallucination, jailbreaking)
```

---

## 🚀 Deployment Specification

### Environment Configuration

```yaml
environments:
  development:
    deployment: Vercel Preview
    database: Neon (branch per PR)
    vector_db: Pinecone (dev namespace)
    observability: Local (OpenTelemetry collector)

  staging:
    deployment: Vercel
    database: Vercel Postgres
    vector_db: Pinecone (staging namespace)
    observability: DataDog

  production:
    deployment: Vercel (with CDN)
    database: Vercel Postgres (HA)
    vector_db: Pinecone (production namespace)
    observability: DataDog + Sentry
```

### CI/CD Pipeline

```yaml
pipeline:
  trigger: [push, pull_request]

  stages:
    1_lint:
      - ESLint
      - TypeScript type check
      - Prettier format check

    2_test:
      - Unit tests (parallel)
      - Integration tests (parallel)

    3_rag_evaluation:
      - Run RAGAS evaluation
      - Compare metrics against baseline
      - Fail if regression > 5%

    4_build:
      - Next.js build
      - Check bundle size (< 500KB first load)

    5_e2e_tests:
      - Playwright tests (staging environment)

    6_deploy:
      - Deploy to Vercel
      - Run smoke tests
      - Notify team

  deployment_strategy:
    type: Blue-Green
    rollback_on_error: automatic
    health_check_endpoint: /api/health
```

---

## 🎨 Code Style & Standards

### TypeScript Standards

```typescript
// MUST: Use strict mode
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true
  }
}

// MUST: Prefer explicit return types for public APIs
export async function generateResponse(
  query: string
): Promise<AgentResponse> {
  // implementation
}

// MUST: Use Zod for runtime validation
const userInputSchema = z.object({
  query: z.string().min(1).max(2000),
  sessionId: z.string().uuid(),
});

// MUST: Document all public interfaces with JSDoc
/**
 * Executes an agentic RAG query with multi-step reasoning.
 *
 * @param query - User's natural language query
 * @param context - Optional conversation context
 * @returns Promise resolving to agent response with sources
 * @throws {ValidationError} If query is invalid
 * @throws {RateLimitError} If rate limit exceeded
 */
```

### Error Handling Standards

```typescript
// MUST: Use custom error classes
export class RagError extends Error {
  constructor(
    message: string,
    public code: string,
    public details?: unknown
  ) {
    super(message);
    this.name = 'RagError';
  }
}

// MUST: Implement circuit breakers for external services
const vectorStoreCircuitBreaker = new CircuitBreaker({
  failureThreshold: 5,
  resetTimeout: 60000,
});

// MUST: Log errors with context
logger.error('Vector search failed', {
  error: err,
  query: redactedQuery,
  userId: user.id,
  traceId: span.traceId,
});
```

---

## 📝 Documentation Standards

### Code Documentation

```typescript
/**
 * SPECIFICATION: Component/Function Name
 *
 * PURPOSE: What is the intent of this component?
 *
 * BEHAVIOR:
 * - Step 1: What happens first
 * - Step 2: What happens next
 * - ...
 *
 * CONSTRAINTS:
 * - Performance: < Xms
 * - Input: Valid range/format
 * - Output: Expected format
 *
 * EXAMPLES:
 * ```typescript
 * const result = await myFunction({ param: 'value' });
 * // result => { expected: 'output' }
 * ```
 */
```

### Living Documentation Updates

```yaml
update_policy:
  when_to_update:
    - Before implementing new features
    - After architectural decisions
    - When APIs change
    - When constraints change

  review_process:
    - Engineer proposes spec update (PR to CLAUDE.md)
    - Team reviews intent and feasibility
    - AI agent validates consistency
    - Merge after approval

  versioning:
    scheme: Semantic versioning (MAJOR.MINOR.PATCH)
    breaking_changes: Require MAJOR version bump
```

---

## 🤖 AI Agent Instructions

### For Claude Code

When working on this project:

1. **ALWAYS** read this CLAUDE.md file first before making changes
2. **VERIFY** your implementation matches the specifications
3. **ASK** for clarification if specification is ambiguous
4. **UPDATE** this document if you discover the specification is incomplete
5. **NEVER** implement features not specified here without asking
6. **PRIORITIZE** intent over implementation details

### Code Generation Rules

```yaml
when_generating_code:
  1_check_specification:
    - Does CLAUDE.md specify this component?
    - What are the CONSTRAINTS?
    - What are the BEHAVIOR requirements?

  2_verify_constraints:
    - Performance requirements met?
    - Security requirements met?
    - Type safety enforced?

  3_add_tests:
    - Unit tests for business logic
    - Integration tests for external dependencies
    - Add test cases to evaluation datasets if RAG-related

  4_update_documentation:
    - Add JSDoc comments
    - Update CHANGELOG.md
    - Update README.md if user-facing
```

### Decision Framework

```yaml
when_making_decisions:
  1_check_specification: Is this decision covered by spec?

  2_if_specified:
    action: Follow specification exactly

  3_if_not_specified:
    action: Ask human for guidance
    context: Explain options and trade-offs

  4_if_ambiguous:
    action: Propose interpretation and ask for confirmation

  5_after_decision:
    action: Update CLAUDE.md with new specification
    commit: "docs: update specification for [feature]"
```

---

## 📈 Metrics & Monitoring

### Key Performance Indicators (KPIs)

```yaml
latency_metrics:
  first_token_latency:
    target: P95 < 3s
    alert_threshold: P95 > 5s

  full_response_latency:
    target: P95 < 10s
    alert_threshold: P95 > 15s

  tool_execution_latency:
    target: P95 < 500ms per tool
    alert_threshold: P95 > 1s

quality_metrics:
  validation_accuracy:
    target: > 95%
    measurement: Against ground truth dataset

  source_attribution:
    target: 100% of factual claims
    measurement: Automated parsing

  user_satisfaction:
    target: > 4.0/5.0
    measurement: Thumbs up/down feedback

cost_metrics:
  cost_per_query:
    target: < $0.10
    breakdown:
      - LLM inference: ~60%
      - Embeddings: ~20%
      - Vector search: ~15%
      - Other: ~5%

reliability_metrics:
  uptime:
    target: > 99.5%

  error_rate:
    target: < 0.5%
```

### Alerting Rules

```yaml
critical_alerts:
  - condition: error_rate > 5%
    action: Page on-call engineer

  - condition: p95_latency > 10s
    action: Slack alert + auto-rollback

  - condition: validation_accuracy < 90%
    action: Disable system + alert team

warning_alerts:
  - condition: cost_per_query > $0.15
    action: Slack alert

  - condition: cache_hit_rate < 50%
    action: Email alert
```

---

## 🔄 Change Management

### Specification Update Process

```yaml
process:
  1_propose:
    - Create PR with CLAUDE.md changes
    - Explain rationale in PR description
    - Tag relevant stakeholders

  2_review:
    - Team reviews intent and feasibility
    - AI agent checks for consistency
    - Discuss trade-offs

  3_approve:
    - Requires 2 approvals
    - Must pass automated checks

  4_implement:
    - Implementation PR references spec PR
    - Implementation must match approved spec
    - Tests must validate spec compliance

  5_verify:
    - Run full test suite
    - Run RAG evaluation
    - Deploy to staging
```

### Version History

```yaml
# Version history tracked in git commits
# Major versions indicate breaking changes
# Minor versions indicate new features
# Patch versions indicate clarifications/fixes

current_version: "1.0.0"
previous_versions:
  - version: "0.1.0"
    date: "2025-11-06"
    changes: "Initial specification"
```

---

## 📚 Reference Documentation

### Related Documents
- [Implementation Roadmap](./enterprise-agentic-rag-research.md)
- [API Documentation](./docs/api.md) - *To be created*
- [Deployment Guide](./docs/deployment.md) - *To be created*
- [Contributing Guide](./CONTRIBUTING.md) - *To be created*

### External Resources
- [Vercel AI SDK Docs](https://sdk.vercel.ai/docs)
- [Specification-Driven Development Guide](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
- [RAGAS Documentation](https://docs.ragas.io/)

---

## ✅ Specification Compliance Checklist

Before merging any PR, verify:

- [ ] Implementation matches CLAUDE.md specification
- [ ] All constraints are enforced
- [ ] Performance requirements are met
- [ ] Security requirements are implemented
- [ ] Tests validate specification compliance
- [ ] Documentation is updated
- [ ] CHANGELOG.md is updated
- [ ] No deviations from specification without approval

---

## 🎯 Current Development Status

```yaml
phase: Planning & Specification
status: Specification Complete

completed:
  - ✅ Project vision defined
  - ✅ Architecture designed
  - ✅ Component specifications written
  - ✅ Technical stack selected
  - ✅ Living specification established

next_steps:
  - [ ] Phase 1: Foundation (Week 1-3)
    - [ ] Project setup
    - [ ] Vector store integration
    - [ ] Basic RAG implementation
  - [ ] Phase 2: Agentic Features (Week 4-7)
    - [ ] ToolLoopAgent implementation
    - [ ] Tool definitions
    - [ ] Multi-agent coordination
  - [ ] Phase 3: Enterprise Features (Week 8-11)
    - [ ] Observability
    - [ ] Security
    - [ ] Performance optimization
  - [ ] Phase 4: Evaluation & Iteration (Week 12+)
    - [ ] Continuous evaluation
    - [ ] CI/CD integration
    - [ ] Production deployment
```

---

## 📞 Contacts & Ownership

```yaml
project_owner: [Your Name/Team]
technical_lead: [Lead Engineer Name]
ai_agent_primary: Claude Code
specification_maintainers: [Core Team]

communication_channels:
  discussions: GitHub Discussions
  issues: GitHub Issues
  urgent: Slack #agentic-rag
```

---

**END OF SPECIFICATION**

*This is a living document. All changes must be tracked in version control.*
*The specification is the source of truth. Code is derived from specification.*

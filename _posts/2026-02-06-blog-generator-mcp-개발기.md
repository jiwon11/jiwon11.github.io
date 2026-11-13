---
title: "[MCP] Blog Generator MCP 개발기 — AI로 기술 블로그를 자동 생성하는 MCP 서버 만들기"
date: 2026-02-06
category: 개발기
tags: [mcp, typescript, gemini, claude, ai, blog, model-context-protocol]
image:
  path: "https://mintcdn.com/mcp/bEUxYpZqie0DsluH/images/mcp-simple-diagram.png?w=1650&fit=max&auto=format&n=bEUxYpZqie0DsluH&q=85&s=a131a609c7b6a70f342f493bbad57fcb"
  thumbnail: "https://mintcdn.com/mcp/bEUxYpZqie0DsluH/images/mcp-simple-diagram.png?w=560&fit=max&auto=format&n=bEUxYpZqie0DsluH&q=85&s=2391513484df96fa7203739dae5e53b0"
---

안녕하세요. 백엔드 개발자 정지원입니다. 요즘은 AI를 활용한 개발 자동화 도구를 만들고 있습니다. 이 글에서는 Blog Generator MCP를 설계한 과정부터 6번의 메이저 버전 업그레이드까지를 정리합니다. 이 글도 MCP의 도움을 받아 작성했습니다.

## 목차

1. 블로그를 쓰고 싶지만
2. MCP란 무엇인가
3. 전체 아키텍처
4. 버전별 변화
5. 개발하며 배운 점
6. MCP 서버 핵심 코드
7. 회고와 앞으로의 계획
8. 참고 자료

## 1. 블로그를 쓰고 싶지만

기술 블로그를 운영하면 개발자에게 여러모로 도움이 됩니다. 아는 내용을 정리해 공유할 수 있고 다른 개발자와 소통할 기회도 생깁니다. 하지만 현실은 녹록지 않습니다.

| 블로그 운영의 어려움 | 구체적 상황 | 결과 |
|---|---|---|
| 시간 부족 | 업무 + 코딩 + 학습으로 글 쓸 여유 없음 | 블로그 방치 |
| 글쓰기 진입 장벽 | 빈 에디터 앞에서 막막함 | 시작조차 못함 |
| 품질 고민 | "이 정도로 올려도 될까?" 고민 | 완성 후에도 게시 망설임 |
| 반복 작업 | frontmatter, 이미지, 태그 등 매번 수동 설정 | 귀찮음 누적 |

> 개발하면서 이미 코드를 작성하고, 의사결정을 내리고, 문제를 해결하고 있습니다. 이 과정을 자동으로 블로그 글로 바꿀 수 있다면 어떨까요?
{: .prompt-tip }

그래서 AI로 기술 블로그 글을 자동으로 만들어 주는 도구를 개발하기로 했습니다. 이것이 Blog Generator MCP의 시작입니다.

![블로그 글감에서 기술 블로그까지](/assets/images/posts/blog-generator-mcp/diagrams/blog-sources.svg){: .light }
![블로그 글감에서 기술 블로그까지](/assets/images/posts/blog-generator-mcp/diagrams/blog-sources-dark.svg){: .dark }
## 2. MCP란 무엇인가

MCP(Model Context Protocol)는 Anthropic에서 공개한 오픈 프로토콜로, AI 모델이 외부 서비스와 상호작용할 수 있도록 표준화된 인터페이스를 제공합니다.

![MCP 클라이언트·서버·외부 서비스 구성](/assets/images/posts/blog-generator-mcp/diagrams/mcp-architecture.svg){: .light }
![MCP 클라이언트·서버·외부 서비스 구성](/assets/images/posts/blog-generator-mcp/diagrams/mcp-architecture-dark.svg){: .dark }
### MCP의 핵심 개념

| 개념 | 설명 | Blog Generator에서의 활용 |
|---|---|---|
| Tool | AI가 호출할 수 있는 함수 | `blog_start_draft`, `blog_save` 등 10개 도구 |
| Transport | 클라이언트-서버 통신 방식 | stdio (로컬) / HTTP (원격) |
| Schema | 도구의 입출력 정의 | Zod 스키마로 타입 안전성 확보 |
| Resource | AI에게 제공하는 데이터 | 작업 상태, 초안 내용 등 |

> MCP는 API를 호출하는 데서 그치지 않고 AI 모델이 도구를 사용하는 방법을 이해할 수 있도록 설계되었습니다. 도구 사용법은 Zod 스키마의 `description` 필드가 AI에게 설명합니다.
{: .prompt-info }

## 3. 전체 아키텍처

MCP는 현재 10개의 도구로 이루어져 있고 Standard Mode와 Pro Mode 두 가지 모드를 제공합니다.

![Standard·Pro 모드 작업 흐름](/assets/images/posts/blog-generator-mcp/diagrams/mode-workflow.svg){: .light }
![Standard·Pro 모드 작업 흐름](/assets/images/posts/blog-generator-mcp/diagrams/mode-workflow-dark.svg){: .dark }
### Standard Mode vs Pro Mode 비교

| 특성 | Standard Mode | Pro Mode |
|---|---|---|
| AI 모델 | Google Gemini (Flash/Pro) | Anthropic Claude Opus |
| 입력 방식 | keyword, code, memo, git_push, notion | code_diff + dev_log |
| 글 스타일 | tutorial, til, deep-dive, troubleshooting | deep-dive 중심 |
| 웹 검색 | 선택적 활성화 | 기본 활성화 |
| 검수 기능 | blog_start_review로 별도 검수 | 작성 시 자체 검수 |
| 비용 | Gemini API 비용 (저렴) | Anthropic API 비용 (고가) |
| 품질 | 양호 | 최상급 |
| 속도 | 빠름 (Flash 기준 30초~1분) | 느림 (2~5분) |

## 4. 버전별 변화

MCP가 처음부터 지금 같은 모습이었던 것은 아닙니다. 여러 번 시행착오를 겪으며 고쳐 온 결과가 현재의 구조입니다.

![Blog Generator MCP 버전 히스토리](/assets/images/posts/blog-generator-mcp/diagrams/version-history.svg){: .light }
![Blog Generator MCP 버전 히스토리](/assets/images/posts/blog-generator-mcp/diagrams/version-history-dark.svg){: .dark }
### v1.0: 4개의 도구로 시작한 기본 생성기

첫 번째 버전에서는 가장 기본적인 파이프라인을 구현했습니다.

![Standard 모드 호출 순서](/assets/images/posts/blog-generator-mcp/diagrams/standard-sequence.svg){: .light }
![Standard 모드 호출 순서](/assets/images/posts/blog-generator-mcp/diagrams/standard-sequence-dark.svg){: .dark }
이 시점에 가장 어려웠던 문제는 오래 걸리는 AI 작업을 어떻게 처리하느냐였습니다. MCP 도구 호출은 기본적으로 동기적이지만 Gemini API 호출은 수십 초가 걸릴 수 있습니다.

### v2.0: 인터랙티브 워크플로우, sql.js, Zod 스키마

v2.0에서는 사용자 경험을 크게 개선했습니다.

| 기능 | 도입 이유 | 구현 방식 |
|---|---|---|
| sql.js | 작업 상태 영속화 필요 | 서버리스 SQLite, 메모리 DB |
| 백그라운드 작업 | AI 호출 대기 시간 해결 | 비동기 실행 + 폴링 |
| Zod 스키마 | 타입 안전성 + AI UX | 입력 검증 + description |
| 인터랙티브 피드백 | 사용자 개입도 향상 | 피드백 히스토리 관리 |
| 다중 입력 유형 | 다양한 글감 지원 | keyword, code, memo, git_push |

상태 저장소 후보를 비교해 sql.js를 선택한 이유는 다음과 같습니다.

![상태 저장소 선택](/assets/images/posts/blog-generator-mcp/diagrams/db-choice.svg){: .light }
![상태 저장소 선택](/assets/images/posts/blog-generator-mcp/diagrams/db-choice-dark.svg){: .dark }
### v3.0: Gemini 분석 + Claude 작성의 Pro Mode

v3.0의 가장 큰 변화는 AI 분업 패턴을 도입한 것이었습니다.

![Pro 모드 호출 순서](/assets/images/posts/blog-generator-mcp/diagrams/pro-sequence.svg){: .light }
![Pro 모드 호출 순서](/assets/images/posts/blog-generator-mcp/diagrams/pro-sequence-dark.svg){: .dark }
> Gemini는 대용량 코드 분석에, Claude는 구조화된 글쓰기에 강합니다. 두 모델의 장점을 조합하니 단일 모델보다 훨씬 높은 품질의 결과물을 얻을 수 있었습니다.
{: .prompt-tip }

### v3.1 ~ v3.2: instructions_file과 환경 변수

| 버전 | 기능 | 해결한 문제 |
|---|---|---|
| v3.1 | `instructions_file` 파라미터 | 매번 스타일 가이드를 복붙하는 번거로움 해소 |
| v3.2 | 환경 변수 지원 | API 키를 파라미터로 전달하는 보안 위험 제거 |

`instructions_file`은 마크다운 파일 경로를 지정하면 해당 파일의 내용을 작성 지침으로 사용합니다. 덕분에 팀 단위로 글쓰기 스타일 가이드를 공유하고 블로그 글의 품질을 일정하게 유지할 수 있게 되었습니다.

### v4.1: Claude Opus 단일 파이프라인

최신 버전에서는 Claude Opus의 성능이 좋아져 분석과 작성을 하나의 모델에서 처리하도록 바꿨습니다.

| 비교 항목 | v3.0 (Gemini + Claude) | v4.1 (Claude Opus 단일) |
|---|---|---|
| API 호출 횟수 | 2회 (분석 + 작성) | 1회 |
| 총 소요 시간 | 3~5분 | 2~3분 |
| 컨텍스트 유실 | 분석→작성 전달 시 일부 유실 | 없음 (단일 컨텍스트) |
| 비용 | Gemini + Claude | Claude만 |
| 글 품질 | 우수 | 최우수 |

## 5. 개발하며 배운 점

### 5.1 Zod 스키마 = MCP UX

Zod 스키마를 어떻게 정의하느냐가 MCP의 사용자 경험을 결정합니다. AI 모델은 스키마의 `description`을 읽고 도구를 어떻게 사용할지 결정하기 때문입니다.

```typescript
// ❌ 나쁜 예 - AI가 이해하기 어려움
const BadSchema = z.object({
  type: z.string(),
  data: z.string(),
});

// ✅ 좋은 예 - AI가 정확하게 이해
const GoodSchema = z.object({
  input_type: z.enum(["keyword", "code", "memo", "git_push", "notion"])
    .describe("입력 유형: keyword(키워드/주제), code(코드 스니펫), memo(메모/노트), git_push(git 변경사항), notion(Notion 페이지 URL)"),
  content: z.string()
    .min(1).max(50000)
    .describe("블로그 글 생성에 사용할 입력 내용"),
  style: z.enum(["tutorial", "til", "deep-dive", "troubleshooting"])
    .default("tutorial")
    .describe("블로그 글 스타일"),
});
```

> `describe()`에 작성하는 설명은 사용자가 아니라 AI 모델이 읽습니다. 어떤 상황에서 이 도구를 사용해야 하는지, 각 파라미터에 어떤 값을 넣어야 하는지 명확히 안내해야 합니다.
{: .prompt-tip }

### 5.2 백그라운드 작업 + 폴링 패턴

AI API 호출은 수십 초가 걸릴 수 있습니다. 그래서 MCP 도구가 호출되면 바로 응답하고 작업은 백그라운드에서 처리한 뒤 폴링으로 상태를 확인하는 패턴을 사용했습니다.

![비동기 작업과 상태 조회](/assets/images/posts/blog-generator-mcp/diagrams/async-task.svg){: .light }
![비동기 작업과 상태 조회](/assets/images/posts/blog-generator-mcp/diagrams/async-task-dark.svg){: .dark }
### 5.3 에러 메시지에 해결 방법 제시

MCP 도구에서 에러가 발생했을 때 에러 내용만 보여 주지 않고 해결 방법까지 함께 제시하면 AI가 스스로 문제를 해결할 수 있습니다.

```typescript
// ❌ 나쁜 에러 메시지
throw new Error("API key invalid");

// ✅ 좋은 에러 메시지 - AI가 사용자에게 안내 가능
throw new Error(
  "Gemini API 키가 유효하지 않습니다. " +
  "다음 방법으로 해결할 수 있습니다:\n" +
  "1. GEMINI_API_KEY 환경변수가 올바르게 설정되어 있는지 확인\n" +
  "2. Google AI Studio에서 새 API 키 발급: https://aistudio.google.com/\n" +
  "3. claude_desktop_config.json의 env 섹션에 키 추가"
);
```

### 5.4 환경 변수의 중요성

| 관리 방식 | 보안 | 편의성 | 적용 |
|---|---|---|---|
| 파라미터 직접 전달 | 낮음 (대화 기록에 노출) | 낮음 (매번 입력) | v1.0~v3.1 |
| 환경 변수 | 높음 (프로세스 내부에서만 접근) | 높음 (한번 설정으로 영구) | v3.2+ |
| .env 파일 | 주의 (실수로 커밋 가능) | 높음 (편리) | 지원 |

## 6. MCP 서버 핵심 코드

### 서버 초기화

MCP 서버는 `@modelcontextprotocol/sdk`로 만듭니다.

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

// 1. 서버 인스턴스 생성
const server = new McpServer({
  name: "blog-generator",
  version: "4.1.0",
});

// 2. 도구 등록
registerStartDraftTool(server);      // blog_start_draft
registerGetStatusTool(server);       // blog_get_status
registerApplyFeedbackTool(server);   // blog_apply_feedback
registerFinalizeDraftTool(server);   // blog_finalize_draft
registerSaveBlogTool(server);        // blog_save
registerDeployGithubTool(server);    // blog_deploy_github
registerStartReviewTool(server);     // blog_start_review
registerStartDraftProTool(server);   // blog_start_draft_pro
registerApplyFeedbackProTool(server);// blog_apply_feedback_pro
registerApplyReviewFeedbackTool(server); // blog_apply_review_feedback

// 3. Transport 연결 및 실행
const transport = new StdioServerTransport();
await server.connect(transport);
```

### 도구 등록 패턴

각 도구는 `server.registerTool()` 또는 `server.tool()`로 등록합니다.

```typescript
import { z } from "zod";

server.tool(
  "blog_start_draft",  // 도구 이름
  {
    // Zod 스키마로 입력 정의
    input_type: z.enum(["keyword", "code", "memo", "git_push", "notion"])
      .describe("입력 유형"),
    content: z.string().min(1).max(50000)
      .describe("블로그 글 생성에 사용할 입력 내용"),
    style: z.enum(["tutorial", "til", "deep-dive", "troubleshooting"])
      .default("tutorial")
      .describe("블로그 글 스타일"),
    model: z.enum(["gemini-1.5-flash", "gemini-1.5-pro", "gemini-2.0-flash"])
      .default("gemini-1.5-flash")
      .describe("사용할 Gemini 모델"),
    web_search: z.boolean().default(false)
      .describe("웹 검색 활용 여부"),
  },
  async (params) => {
    // 1. 작업 생성
    const taskId = generateTaskId();
    await createTask(taskId, params);

    // 2. 백그라운드 실행
    generateDraftInBackground(taskId, params);

    // 3. 즉시 응답
    return {
      content: [{
        type: "text",
        text: JSON.stringify({
          task_id: taskId,
          status: "pending",
          message: "블로그 생성이 시작되었습니다."
        })
      }]
    };
  }
);
```

### 프로젝트 구조

```
blog-generator-mcp/
├── src/
│   ├── index.ts              # 서버 진입점
│   ├── types.ts              # Zod 스키마 + 타입 정의
│   ├── tools/
│   │   ├── startDraft.ts     # Standard Mode 초안 생성
│   │   ├── startDraftPro.ts  # Pro Mode 초안 생성
│   │   ├── getStatus.ts      # 작업 상태 조회
│   │   ├── applyFeedback.ts  # 피드백 반영
│   │   ├── finalizeDraft.ts  # 초안 확정
│   │   ├── saveBlog.ts       # 로컬 저장
│   │   ├── deployGithub.ts   # GitHub 배포
│   │   └── startReview.ts    # 검수
│   └── services/
│       ├── database.ts       # sql.js DB 관리
│       ├── gemini.ts         # Gemini API 클라이언트
│       ├── claude.ts         # Claude API 클라이언트
│       ├── github.ts         # GitHub API 클라이언트
│       ├── env.ts            # 환경 변수 관리
│       ├── instructions.ts   # 작성 지침 로더
│       └── thumbnail.ts      # 썸네일 이미지 검색
├── data/
│   └── tasks.db              # sql.js 데이터베이스
├── posts/                    # 생성된 블로그 글 저장
├── example-instructions.md   # 기본 작성 지침
├── package.json
└── tsconfig.json
```

## 7. 회고와 앞으로의 계획

Blog Generator MCP는 "블로그 쓰기가 귀찮다"는 단순한 불편함에서 시작했습니다. 6번의 메이저 버전 업그레이드를 거쳐 지금은 10개의 도구를 갖춘 AI 블로그 자동화 시스템이 되었습니다.

### 배운 점

| 학습 포인트 | 내용 |
|---|---|
| MCP 설계 | Zod 스키마의 description이 AI UX를 결정한다 |
| AI 분업 | 각 모델의 강점을 조합하면 단일 모델보다 나은 결과를 얻는다 |
| 비동기 패턴 | 백그라운드 작업 + 폴링이 장시간 AI 작업의 해법이다 |
| 에러 설계 | 에러 메시지에 해결 방법을 포함하면 AI가 자동 복구할 수 있다 |
| 환경 변수 | 민감 정보는 반드시 환경 변수로 관리해야 한다 |
| 지침 파일 | 글쓰기 스타일을 파일로 외부화하면 품질 일관성을 유지할 수 있다 |

### 향후 계획

![v5.0 계획](/assets/images/posts/blog-generator-mcp/diagrams/v5-roadmap.svg){: .light }
![v5.0 계획](/assets/images/posts/blog-generator-mcp/diagrams/v5-roadmap-dark.svg){: .dark }
- 다중 AI 모델 통합: OpenAI GPT, Google Gemini, Anthropic Claude를 상황에 맞게 자동 선택
- 이미지 자동 생성: DALL-E, Midjourney 등을 활용한 블로그 삽화 자동 생성
- 시리즈 글 관리: 연재물의 이전/다음 편 자동 연결 및 목차 관리
- SEO 자동 최적화: 키워드 분석, 메타 태그 자동 생성, 내부 링크 추천

## 8. 참고 자료

### 공식 문서

- [Model Context Protocol](https://modelcontextprotocol.io/): MCP 공식 사양
- [Zod 공식 문서](https://zod.dev/): TypeScript 스키마 검증
- [Google AI Platform](https://ai.google.dev/): Gemini API

### 기술 블로그

- [sql.js GitHub 저장소](https://github.com/sql-js/sql.js): 브라우저/Node.js용 SQLite
- [Anthropic API Documentation](https://docs.anthropic.com/): Claude API

### 튜토리얼 가이드

- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk): MCP 서버 구축 SDK

---

> 이 글은 Blog Generator MCP의 도움을 받아 작성했습니다. 궁금한 점이나 피드백이 있으면 댓글로 남겨 주세요.

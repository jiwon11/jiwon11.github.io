---
title: "AI 대화 후처리 개선기 — 응답 트랜잭션에서 요약·친밀도·보상을 분리하기까지"
date: 2026-06-29
category: 개발기
tags: [spring-boot, kotlin, coroutine, event-driven, openai, transaction, architecture]
readTime: 15min
mermaid: true
image:
  path: /assets/images/posts/ai-conversation-post-processing/network-cables.jpg
  thumbnail: /assets/images/posts/ai-conversation-post-processing/network-cables.jpg
---

# AI 대화 후처리 개선기 — 응답 트랜잭션에서 요약·친밀도·보상을 분리하기까지

안녕하세요. duurian 팀에서 백엔드 개발을 담당하고 있는 정지원입니다.

AI 대화의 마지막 턴에는 사용자에게 답변을 돌려주는 일 외에도 많은 작업이 따라옵니다. 대화를 요약하고, 대화 품질을 판별하고, 친밀도를 갱신하고, 조건에 맞으면 보상을 생성해야 합니다. 처음에는 이 작업들을 하나의 요청 흐름에서 처리했습니다. 구현은 단순했지만 사용자는 자신의 응답과 직접 관계없는 후처리가 끝날 때까지 기다려야 했습니다.

이번 글은 마지막 턴의 저장과 응답만 동기 경로에 남기고, 요약·친밀도·보상을 커밋 이후의 독립된 파이프라인으로 옮긴 과정을 다룹니다. 단순히 “코루틴으로 비동기화했다”는 이야기가 아니라 다음 질문에 답하는 개발기입니다.

- 후처리는 언제 시작해야 커밋된 대화를 안전하게 읽을 수 있는가?
- AI 호출과 DB 트랜잭션은 어디에서 분리해야 하는가?
- 요약, 친밀도, 보상 사이의 순서는 어떻게 보장할 것인가?
- 실패와 애플리케이션 종료까지 포함하면 현재 구조는 어디까지 안전한가?

> 내부 성능 측정값과 세부 보상 트리거는 공개용으로 일반화했습니다. 표와 다이어그램은 절대 시간 대신 상대적인 전후 관계를 보여줍니다.
{: .prompt-info }

![서버의 연결과 데이터 흐름](/assets/images/posts/ai-conversation-post-processing/network-cables.jpg)

<small>후처리 파이프라인을 떠올리게 하는 네트워크 연결. Photo by [Brett Sayles on Pexels](https://www.pexels.com/photo/cables-connected-on-server-2881229/).</small>

<div class="notice--info" markdown="1">
**AI 대화와 보상 파이프라인 개발기**

1. **AI 대화 후처리 개선기** — 현재 글
2. [리워드 생성·수령·충전 책임 분리](/posts/reward-creation-receipt-charge-responsibility/)
3. [상태 조건 UPDATE로 중복 수령 방지](/posts/reward-conditional-update-idempotency/)
</div>

---

## 1. 한눈에 보는 개선 결과

| 구분 | 개선 전 | 개선 후 |
|---|---|---|
| 사용자 응답 경로 | 대화 저장 + AI 요약까지 대기 | 사용자/AI 대화 저장까지만 담당 |
| 후처리 시작점 | 요청 흐름 내부 | 대화 트랜잭션 커밋 이후 |
| AI 호출과 DB | 긴 흐름 안에서 혼재 | AI 호출은 트랜잭션 밖, 저장만 짧은 트랜잭션 |
| 작업 순서 | 호출 위치에 암묵적으로 의존 | 요약 → 친밀도 → 보상으로 명시 |
| 작업 격리 | 한 실패가 전체 요청에 영향 | 사용자별 작업과 단계별 예외를 격리 |
| 체감 성능 | 후처리까지 모두 대기 | 임계 경로가 크게 단축 |

초기 테스트에서는 “사용자가 기다릴 이유가 없는 구간”을 응답 경로에서 제거한 뒤 마지막 턴의 체감 지연이 크게 줄었습니다.

~~~mermaid
flowchart LR
    subgraph before["개선 전 · 후처리까지 대기"]
        B1["대화 생성"] --> B2["대화 저장"]
        B2 --> B3["AI 요약"]
        B3 --> B4["요약 저장"]
        B4 --> B5["HTTP 응답"]
    end

    subgraph after["개선 후 · 임계 경로 단축"]
        A1["대화 생성"] --> A2["대화 저장"]
        A2 --> A3["HTTP 응답"]
        A2 -. "커밋 후 이벤트" .-> A4["요약·친밀도·보상"]
    end

    style B5 fill:#ef9a9a,stroke:#c62828,color:#111
    style A3 fill:#a5d6a7,stroke:#2e7d32,color:#111
    style A4 fill:#bbdefb,stroke:#1565c0,color:#111
~~~

---

## 2. 문제의 시작: 마지막 턴이 너무 많은 책임을 가졌다

일반적인 대화 턴은 “입력 → AI 응답 → 저장 → 반환”으로 끝납니다. 하지만 마지막 턴은 달랐습니다.

~~~mermaid
flowchart TD
    T["마지막 대화 턴"] --> C["사용자·AI 대화 저장"]
    T --> S["오늘 대화 요약 생성"]
    T --> Q["대화 품질 판별"]
    T --> F["친밀도 갱신"]
    T --> R["조건에 맞는 보상 생성"]

    S --> Q
    S --> F
    Q --> R
    F --> R
~~~

이 작업들을 요청 안에서 순서대로 실행하면 다음 문제가 생깁니다.

1. OpenAI 응답 시간이 그대로 마지막 턴 지연에 더해집니다.
2. 요약 실패가 이미 성공한 대화 자체를 실패처럼 보이게 만들 수 있습니다.
3. 외부 API를 기다리는 동안 DB 트랜잭션을 열어둘 유혹이 생깁니다.
4. 친밀도와 보상이 “커밋되지 않은 대화”를 읽는 타이밍 문제가 생깁니다.

<div class="notice--warning" markdown="1">
**핵심 문제**

대화 저장은 사용자가 요청한 핵심 기능이고, 요약·친밀도·보상은 그 결과를 소비하는 후처리입니다. 서로 실패 의미와 지연 허용 범위가 다른데 같은 생명주기에 묶여 있었습니다.
</div>

---

## 3. 먼저 동기 경로의 완료 조건을 다시 정의했다

최적화의 출발점은 기술 선택이 아니라 완료 조건의 재정의였습니다.

> “사용자와 AI의 대화가 DB에 안전하게 저장되면 이 HTTP 요청은 성공이다.”

요약이나 보상 생성은 즉시 끝날 필요가 없습니다. 반면 저장되지 않은 대화를 기반으로 요약을 만들면 안 됩니다. 그래서 트랜잭션을 두 종류로 나눴습니다.

| 트랜잭션 | 책임 | 외부 API 포함 | 완료가 사용자 응답을 막는가 |
|---|---|---:|---:|
| T1 | 사용자·AI 대화 저장 | 아니오 | 예 |
| T2 | 생성된 요약 저장 | 아니오 | 아니오 |
| 후속 서비스 트랜잭션 | 친밀도·보상 생성 | 아니오 | 아니오 |

~~~mermaid
sequenceDiagram
    autonumber
    actor U as 사용자
    participant API as Conversation API
    participant DB as PostgreSQL
    participant E as Event Listener
    participant AI as OpenAI
    participant P as 후처리 서비스

    U->>API: 마지막 턴 요청
    API->>DB: 사용자·AI 대화 저장 (T1)
    API->>API: ConversationCompletedEvent 발행
    DB-->>API: COMMIT
    API-->>U: 대화 응답

    Note over E,P: T1 커밋 이후, 응답 경로 밖
    E->>AI: 오늘 대화 요약 요청
    AI-->>E: 요약 결과
    E->>DB: 요약 저장 (짧은 T2)
    E->>P: 친밀도 갱신
    E->>P: 보상 생성 시도
~~~

이 구조에서 중요한 점은 “이벤트 발행”과 “이벤트 처리”의 시점이 다르다는 것입니다. 이벤트는 T1 안에서 발행하지만, 리스너는 T1이 성공적으로 커밋된 뒤 실행합니다.

---

## 4. 대화 저장과 이벤트 발행을 한 트랜잭션에 묶었다

현재 구현에서 <code>ConversationPersistenceService</code>는 두 대화를 저장하고 마지막 턴일 때만 완료 이벤트를 발행합니다.

~~~kotlin
@Service
class ConversationPersistenceService(
    private val commandConversationPort: CommandConversationPort,
    private val eventPublisher: ApplicationEventPublisher,
) {

    @Transactional
    fun saveConversationsAndPublishEvent(
        userConversation: Conversation,
        aiConversation: Conversation,
        isLastTurn: Boolean,
        userId: UUID,
    ): Pair<Conversation, Conversation> {
        val savedUser = commandConversationPort.saveConversation(userConversation)
        val savedAi = commandConversationPort.saveConversation(aiConversation)

        if (isLastTurn) {
            eventPublisher.publishEvent(ConversationCompletedEvent(userId))
        }

        return savedUser to savedAi
    }
}
~~~

이벤트를 저장 성공 이후에 직접 호출하는 것처럼 보이지만, 실제 리스너는 <code>AFTER_COMMIT</code>에 결합했습니다.

~~~kotlin
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
fun handleConversationCompleted(event: ConversationCompletedEvent) {
    conversationPostTurnScope.launch(Dispatchers.IO) {
        processPostTurn(event.userId)
    }
}
~~~

Spring 공식 문서에서 <code>AFTER_COMMIT</code>은 트랜잭션이 성공적으로 커밋된 후 리스너를 실행하는 단계입니다. 따라서 저장이 롤백되면 후처리도 시작되지 않습니다.

~~~mermaid
stateDiagram-v2
    [*] --> T1진행
    T1진행 --> 커밋: 저장 성공
    T1진행 --> 롤백: 저장 실패
    커밋 --> 후처리이벤트
    롤백 --> 종료
    후처리이벤트 --> 종료
~~~

<div class="notice--info" markdown="1">
**왜 일반 이벤트 리스너가 아니었을까?**

대화 요약은 오늘 저장된 대화를 다시 조회합니다. 커밋 전 리스너가 먼저 실행되면 같은 데이터가 보인다는 보장이 약해집니다. 후처리 시작 조건을 “메서드가 호출됨”이 아니라 “대화가 커밋됨”으로 잡았습니다.
</div>

---

## 5. 이벤트 리스너와 비동기 실행은 서로 다른 책임이다

<code>AFTER_COMMIT</code>은 실행 시점을 정할 뿐, 작업을 자동으로 비동기화하지 않습니다. 리스너 안에서 오래 걸리는 요약을 그대로 실행하면 커밋 이후라도 요청 스레드는 그 작업을 기다릴 수 있습니다.

그래서 후처리 전용 <code>CoroutineScope</code>를 빈으로 두었습니다.

~~~kotlin
@Configuration
class CoroutineConfig {
    private val supervisorJob = SupervisorJob()

    private val exceptionHandler = CoroutineExceptionHandler { _, exception ->
        log.error(exception) {
            "대화 후처리 코루틴 실행 중 처리되지 않은 예외 발생"
        }
    }

    @Bean
    fun conversationPostTurnScope(): CoroutineScope =
        CoroutineScope(
            supervisorJob +
                Dispatchers.Default +
                exceptionHandler +
                CoroutineName("ConversationPostTurnScope")
        )

    @PreDestroy
    fun cleanup() {
        supervisorJob.cancel("애플리케이션 종료")
    }
}
~~~

각 구성 요소의 책임은 다음과 같습니다.

| 구성 요소 | 담당하는 문제 |
|---|---|
| <code>AFTER_COMMIT</code> | 커밋된 대화만 후처리 |
| <code>launch(Dispatchers.IO)</code> | 요청 경로에서 분리하고 블로킹 I/O용 실행 문맥 사용 |
| <code>SupervisorJob</code> | 한 사용자의 작업 실패가 다른 사용자의 작업을 취소하지 않도록 격리 |
| <code>CoroutineExceptionHandler</code> | 처리되지 않은 루트 코루틴 예외 기록 |
| <code>CoroutineName</code> | 로그와 디버깅에서 후처리 작업 식별 |
| <code>@PreDestroy</code> | 애플리케이션 종료 시 새 작업과 진행 작업 취소 |

~~~mermaid
flowchart TB
    Scope["ConversationPostTurnScope"]
    Scope --> A["사용자 A 후처리"]
    Scope --> B["사용자 B 후처리"]
    Scope --> C["사용자 C 후처리"]
    A --> AF["요약 실패"]
    B --> BS["정상 완료"]
    C --> CS["정상 완료"]
    AF -. "SupervisorJob: 실패 미전파" .- BS
    AF -. "SupervisorJob: 실패 미전파" .- CS

    style AF fill:#ffcdd2,stroke:#c62828,color:#111
    style BS fill:#c8e6c9,stroke:#2e7d32,color:#111
    style CS fill:#c8e6c9,stroke:#2e7d32,color:#111
~~~

<code>SupervisorJob</code>은 “서로 다른 이벤트 작업”을 격리합니다. 한 이벤트 안의 요약·친밀도·보상 순서까지 자동으로 격리하는 것은 아닙니다. 그 부분은 서비스 코드에서 직접 정의했습니다.

---

## 6. 후처리 순서를 코드로 드러냈다

요약, 친밀도, 보상은 완전히 독립적이지 않습니다.

- 친밀도 계산은 오늘 생성된 요약 개수를 사용합니다.
- 저품질 대화 판별은 요약 내용을 사용합니다.
- 보상은 대화 종료 기록과 요약 존재 여부를 확인합니다.

그래서 세 작업을 무조건 병렬로 실행하지 않았습니다.

~~~kotlin
private suspend fun processPostTurn(userId: UUID) {
    createSummary(userId) ?: return

    withContext(NonCancellable) {
        updateFriendship(userId)
        createReward(userId)
    }
}
~~~

~~~mermaid
flowchart TD
    S["1. 요약 생성"] --> OK{성공?}
    OK -->|아니오| STOP["후속 작업 중단"]
    OK -->|예| NC["NonCancellable 구간"]
    NC --> F["2. 친밀도 갱신"]
    F --> R["3. 보상 생성 시도"]
    F -. "실패는 로깅 후 계속" .-> R

    style STOP fill:#ffcdd2,stroke:#c62828,color:#111
    style NC fill:#fff9c4,stroke:#f9a825,color:#111
    style R fill:#c8e6c9,stroke:#2e7d32,color:#111
~~~

여기서 <code>NonCancellable</code>은 친밀도와 보상을 “절대 실패하지 않게” 만들지 않습니다. 부모 코루틴의 취소 신호가 들어왔을 때 해당 블록이 취소로 중단되는 것을 막을 뿐입니다. 각 단계에서 발생한 일반 예외는 여전히 직접 처리해야 합니다.

~~~kotlin
private fun updateFriendship(userId: UUID) {
    try {
        conversationFriendshipService
            .tryUpdateFriendshipAfterConversation(userId)
    } catch (e: Exception) {
        log.error(e) { "친밀도 업데이트 실패" }
    }
}

private fun createReward(userId: UUID) {
    try {
        conversationRewardService
            .tryCreateConversationReward(userId)
    } catch (e: Exception) {
        log.error(e) { "대화 보상 생성 실패" }
    }
}
~~~

### 실패 정책을 표로 고정하기

| 실패 지점 | 대화 저장 | 요약 | 친밀도 | 보상 | 현재 동작 |
|---|---:|---:|---:|---:|---|
| T1 롤백 | 실패 | 미실행 | 미실행 | 미실행 | 이벤트 소비 안 함 |
| AI 요약 호출 실패 | 성공 | 실패 | 미실행 | 미실행 | 로그 후 종료 |
| 요약 저장 실패 | 성공 | 실패 | 미실행 | 미실행 | T2 롤백 후 종료 |
| 친밀도 갱신 실패 | 성공 | 성공 | 실패 | 실행 | 단계 예외 격리 |
| 보상 생성 실패 | 성공 | 성공 | 성공 가능 | 실패 | 로그만 남김 |
| JVM 비정상 종료 | 성공 | 유실 가능 | 유실 가능 | 유실 가능 | 재시도 없음 |

이 표는 구현 설명보다 더 중요했습니다. “비동기”라는 한 단어로는 어느 실패가 어디까지 전파되는지 알 수 없기 때문입니다.

---

## 7. AI 호출을 트랜잭션 밖으로 밀어냈다

요약 생성에는 네 종류의 작업이 섞여 있습니다.

~~~mermaid
flowchart LR
    Q["대화·프롬프트 조회"] --> C["프롬프트 구성"]
    C --> AI["OpenAI 호출"]
    AI --> P["응답 파싱"]
    P --> T2["요약 저장 T2"]

    style AI fill:#bbdefb,stroke:#1565c0,color:#111
    style T2 fill:#c8e6c9,stroke:#2e7d32,color:#111
~~~

가장 오래 걸리고 실패 가능성이 높은 것은 외부 AI 호출입니다. 이 시간을 DB 트랜잭션에 포함하면 커넥션과 트랜잭션 수명이 외부 시스템의 응답 시간에 종속됩니다. 현재 코드는 AI 호출을 먼저 끝내고, 저장 구간만 <code>TransactionTemplate</code>로 감쌉니다.

~~~kotlin
override suspend fun createConversationSummary(
    command: CreateConversationSummaryCommand
): List<String> {
    val conversations =
        queryConversationPort.findAllByUserIdAndCreatedAtInRange(
            command.userId,
            TimezoneConverter.calculateTimeRangeInUtc()
        )

    val generationResult = withContext(Dispatchers.IO) {
        handleSummary(conversations)
    }

    withContext(Dispatchers.IO) {
        transactionalRunner.executeWithoutResult {
            generationResult.parsedSummaries.forEach { content ->
                commandConversationSummaryPort.saveConversationSummary(
                    ConversationSummary.create(command.userId, content)
                )
            }
        }
    }

    return generationResult.parsedSummaries
}
~~~

~~~kotlin
@Component
class TransactionalRunner(
    private val transactionTemplate: TransactionTemplate,
) {
    fun executeWithoutResult(block: () -> Unit) {
        transactionTemplate.executeWithoutResult { block() }
    }
}
~~~

~~~mermaid
gantt
    title 트랜잭션을 짧게 유지한 후처리
    dateFormat  X
    axisFormat %L
    section 외부 작업
    대화 조회·프롬프트 구성 :0, 2
    OpenAI 호출              :2, 9
    응답 파싱                :9, 10
    section DB 트랜잭션
    요약 저장 T2             :10, 12
~~~

Gantt의 숫자는 실제 초가 아니라 상대적인 구간입니다. 핵심은 느린 외부 호출 구간에 DB 트랜잭션이 열려 있지 않다는 점입니다.

---

## 8. 친밀도와 보상 서비스에도 방어 조건을 중복 배치했다

파이프라인이 순서를 보장하더라도 각 서비스는 독립적으로 안전해야 합니다. 이벤트가 중복 호출되거나 대화 종료 API와 후처리가 앞뒤로 교차할 수 있기 때문입니다.

### 친밀도 갱신 전 확인

~~~mermaid
flowchart TD
    A["친밀도 갱신 요청"] --> B{오늘 이벤트가 이미 있는가?}
    B -->|예| X["스킵"]
    B -->|아니오| C{대화 종료 기록이 있는가?}
    C -->|아니오| X
    C -->|예| D{오늘 요약이 있는가?}
    D -->|아니오| X
    D -->|예| E["친밀도 갱신"]
    E --> F{레벨 상승?}
    F -->|예| G["레벨업 보상 생성"]
    F -->|아니오| H["종료"]
~~~

### 대화 보상 생성 전 확인

| 순서 | 가드 조건 | 막는 문제 |
|---:|---|---|
| 1 | 오늘 대화 보상이 이미 존재 | 이벤트 중복 처리 |
| 2 | 오늘 미지급 이력이 이미 존재 | 저품질 판정 재실행 |
| 3 | 대화 종료 기록 존재 | 아직 사용자가 대화를 끝내지 않은 상태 |
| 4 | 오늘 요약 존재 | 후처리 순서 역전 |
| 5 | 저품질 판별 성공 | 판별 실패를 정상 보상으로 오인 |

후처리 오케스트레이터 한 곳만 믿지 않고, 실제 상태를 변경하는 서비스도 자신의 선행 조건을 검증합니다. 이중 방어는 비동기 흐름에서 특히 유용합니다.

---

## 9. 관측 가능성도 기능의 일부로 취급했다

비동기 후처리는 사용자 응답과 분리되므로 실패가 사용자에게 바로 드러나지 않습니다. 로그와 알림이 없으면 “응답은 성공했는데 요약이 없다”는 데이터만 남습니다.

현재 요약 서비스는 다음 메타데이터를 남깁니다.

| 관측 항목 | 목적 |
|---|---|
| 사용자 ID | 영향 사용자 추적 |
| 오늘 대화 수 | 입력 데이터 존재 여부 확인 |
| 파싱 결과 수·빈 값 수 | 모델 응답과 파서 문제 구분 |
| 모델명 | 모델 설정별 장애 비교 |
| 입력·출력·전체 토큰 | 잘림과 비용 이상 징후 확인 |
| AI 원본 응답 일부 | 파싱 실패 원인 확인 |
| 대화 컨텍스트 일부 | 입력 구성 문제 확인 |

빈 요약을 감지하면 경고 로그뿐 아니라 별도 알림 포트로 운영 채널에 전달합니다.

~~~mermaid
flowchart LR
    AI["AI 응답"] --> P["파싱"]
    P --> B{유효한 요약이 있는가?}
    B -->|예| S["DB 저장"]
    B -->|아니오| L["구조화 로그"]
    B -->|아니오| N["운영 채널 알림"]
    L --> S
    N --> S
~~~

현재 구현은 빈 요약도 그대로 저장할 수 있습니다. 알림은 탐지를 강화하지만 데이터 정책을 바꾸지는 않습니다. 빈 결과를 저장하지 않을지, 재시도할지는 별도의 제품·운영 정책으로 남겨두었습니다.

---

## 10. 성능 개선을 어떻게 해석해야 하는가

초기 측정에서는 마지막 턴 API의 체감 지연이 크게 줄었습니다.

| 지표 | 개선 전 | 개선 후 |
|---|---|---|
| 마지막 턴 API | 후처리 완료까지 대기 | 대화 저장 뒤 응답 |
| 사용자 대기 경로 | 대화 + 요약 | 대화 생성·저장 |

~~~mermaid
flowchart LR
    B["개선 전<br/><b>후처리까지 대기</b>"] --> D["임계 경로 단축"]
    D --> A["개선 후<br/><b>저장 뒤 응답</b>"]
    style B fill:#ffcdd2,stroke:#c62828,color:#111
    style D fill:#fff9c4,stroke:#f9a825,color:#111
    style A fill:#c8e6c9,stroke:#2e7d32,color:#111
~~~

이 결과를 “AI 요약 자체가 빨라졌다”고 해석하면 안 됩니다. 요약 자체의 실행 시간은 사라진 것이 아니라 사용자 응답의 임계 경로 밖으로 이동했습니다.

> 최적화한 것은 후처리 작업의 계산량이 아니라 사용자가 기다리는 경로의 길이였습니다.

---

## 11. 현재 구조가 보장하지 않는 것

이번 구조는 응답 시간과 트랜잭션 경계를 개선했지만 메시지 브로커가 있는 내구성 파이프라인은 아닙니다.

### 11.1 인메모리 이벤트는 유실될 수 있다

T1 커밋 직후 JVM이 종료되면 대화는 저장됐지만 후처리는 시작되지 않을 수 있습니다. 애플리케이션 이벤트와 코루틴은 프로세스 메모리 안에 있기 때문입니다.

### 11.2 종료 시 취소만 하고 완료를 기다리지 않는다

<code>@PreDestroy</code>는 <code>SupervisorJob.cancel()</code>을 호출하지만 진행 중인 작업의 완료를 기다리는 <code>join</code> 단계는 없습니다. 더구나 친밀도·보상 구간은 <code>NonCancellable</code>입니다. 종료 유예 시간과 작업 완료 정책을 함께 설계해야 합니다.

### 11.3 자동 재시도와 DLQ가 없다

AI 일시 장애나 DB 순간 장애가 발생하면 로그는 남지만 자동 재시도되지 않습니다. 운영자가 탐지한 뒤 수동 보정해야 합니다.

### 11.4 취소 예외를 일반 예외로 다룰 수 있다

요약 생성의 <code>catch (Exception)</code>은 <code>CancellationException</code>까지 잡을 수 있습니다. 코루틴 취소 의미를 보존하려면 취소 예외는 다시 던지는 정책이 더 명확합니다.

~~~mermaid
flowchart TD
    C["대화 T1 커밋"] --> W{"JVM 생존?"}
    W -->|예| P["인메모리 후처리"]
    W -->|아니오| L["이벤트 유실 가능"]
    P --> O{"후처리 성공?"}
    O -->|예| E["완료"]
    O -->|아니오| M["로그·알림"]
    M --> R["자동 재시도 없음"]

    style L fill:#ffcdd2,stroke:#c62828,color:#111
    style R fill:#ffcdd2,stroke:#c62828,color:#111
~~~

---

## 12. 다음 단계: 빠른 비동기화에서 내구성 있는 작업으로

요구되는 신뢰 수준이 높아진다면 다음 단계는 Transactional Outbox와 작업 상태 테이블입니다.

~~~mermaid
sequenceDiagram
    autonumber
    participant API as Conversation API
    participant DB as PostgreSQL
    participant P as Outbox Publisher
    participant Q as Message Queue
    participant W as Post-process Worker

    API->>DB: 대화 + outbox 레코드 저장 (한 트랜잭션)
    DB-->>API: COMMIT
    API-->>API: 사용자 응답
    P->>DB: 미발행 outbox 조회
    P->>Q: ConversationCompleted 발행
    Q->>W: at-least-once 전달
    W->>DB: 단계별 멱등 처리
    W-->>Q: ACK
~~~

| 단계 | 지금 | 다음 설계 |
|---|---|---|
| 이벤트 보존 | JVM 메모리 | DB outbox |
| 전달 | 한 번 시도 | at-least-once |
| 재시도 | 없음 | 지수 백오프 |
| 실패 격리 | 로그 | DLQ 또는 실패 상태 |
| 진행 상태 | 결과 데이터로 추론 | PENDING / RUNNING / SUCCEEDED / FAILED |
| 중복 방지 | 서비스별 사전 조회 | 단계별 멱등 키·제약 |

모든 서비스가 처음부터 메시지 큐를 필요로 하지는 않습니다. 이번 단계에서는 사용자 지연을 크게 줄이는 단순한 구조가 충분했습니다. 다만 현재 보장 범위를 문서화했기 때문에, 요구 수준이 바뀌었을 때 어디를 확장해야 하는지도 분명해졌습니다.

---

## 13. 배운 점

### 13.1 비동기화 전에 완료 조건부터 분리해야 한다

무엇을 백그라운드로 보낼지 결정하려면 먼저 “이 요청의 성공은 무엇인가?”를 정의해야 합니다. 대화 저장이 성공 조건이라는 합의가 있었기 때문에 요약을 안전하게 응답 경로 밖으로 옮길 수 있었습니다.

### 13.2 커밋 시점과 실행 스레드는 별개의 문제다

<code>AFTER_COMMIT</code>은 데이터 가시성 문제를, 별도 코루틴 스코프는 응답 지연 문제를 해결합니다. 둘 중 하나만 적용하면 목표를 완성할 수 없습니다.

### 13.3 느린 외부 호출을 트랜잭션으로 감싸지 않는다

AI 호출이 끝난 뒤 저장할 결과가 생겼을 때만 짧은 트랜잭션을 열었습니다. 이 경계가 커넥션 점유 시간과 장애 전파 범위를 줄였습니다.

### 13.4 실패 정책은 코드보다 표가 더 잘 설명한다

요약 실패 시 보상을 만들 것인지, 친밀도 실패 후 보상을 계속할 것인지 같은 결정은 기술 세부사항이 아니라 제품 정책입니다. 실패 매트릭스로 드러내야 팀이 같은 시스템을 상상할 수 있습니다.

### 13.5 비동기는 완료가 아니라 책임의 이동이다

사용자의 대기 시간은 줄었지만 후처리의 실패 가능성이 사라진 것은 아닙니다. 실패를 누가 탐지하고, 재시도하고, 보정할지까지 정해야 비동기 기능이 운영 가능한 기능이 됩니다.

---

## 마무리

이번 개선의 핵심은 코루틴 자체가 아니었습니다. 대화 저장, 커밋 이후 실행, AI 호출, 짧은 DB 저장, 친밀도와 보상이라는 서로 다른 책임에 각자의 시간과 실패 경계를 준 것이 핵심이었습니다.

그 결과 사용자는 마지막 턴에서 요약이 끝날 때까지 기다리지 않게 됐고, 후처리는 커밋된 대화를 기준으로 명확한 순서와 실패 정책을 갖게 됐습니다. 동시에 인메모리 이벤트 유실, 자동 재시도 부재, 종료 처리 같은 다음 과제도 선명해졌습니다.

성능 최적화는 단순히 더 빠른 코드를 쓰는 일이 아닙니다. 사용자가 기다려야 하는 일과 기다리지 않아도 되는 일을 정확히 가르는 일에서 시작합니다.

---

## 참고 자료

- [Spring Framework — Transaction-bound Events](https://docs.spring.io/spring-framework/reference/6.2/data-access/transaction/event.html)
- [Kotlin — Coroutine exceptions handling](https://kotlinlang.org/docs/exception-handling.html)
- [Kotlin — Coroutine context and dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)
- [kotlinx.coroutines API — NonCancellable과 SupervisorJob](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/)
- [Pexels 이미지 원문 — Brett Sayles](https://www.pexels.com/photo/cables-connected-on-server-2881229/)

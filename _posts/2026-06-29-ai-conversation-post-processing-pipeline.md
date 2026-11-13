---
title: "AI 대화 후처리를 요청 트랜잭션에서 분리하기"
date: 2026-06-29
category: 개발기
tags: [spring-boot, kotlin, coroutine, event-driven, openai, transaction, architecture]
readTime: 15min
image:
  path: /assets/images/posts/ai-conversation-post-processing/network-cables.jpg
  thumbnail: /assets/images/posts/ai-conversation-post-processing/network-cables.jpg
  alt: "비동기 데이터 흐름을 떠올리게 하는 네트워크 연결. Photo by Brett Sayles on Pexels."
---

AI 요청을 처리한 후에는 추가 분석이나 데이터 갱신처럼 사용자 응답과 바로 관계없는 작업이 이어질 수 있습니다. 이 작업을 모두 같은 요청에서 실행하면 외부 API를 기다리는 시간까지 응답이 늦어집니다.

이번 글에서는 요청 데이터가 저장된 후 코루틴으로 후처리를 실행하고 외부 API 호출과 저장 트랜잭션을 나누는 과정을 정리해보겠습니다.

> 아래 이벤트 이름, Task A·B·C, 코드와 다이어그램은 설명을 위해 만든 예제입니다. 실제 실행 조건, 처리 순서, 프롬프트, 모델, 재시도와 운영 규칙은 포함하지 않았습니다.
{: .prompt-info }

> **연관 개발기**
>
> 1. [중복 리워드 처리를 막기 위해 상태 조건 UPDATE를 선택한 이유](/posts/reward-conditional-update-idempotency/)
> 2. [리워드 생성·수령·충전 책임을 분리한 과정](/posts/reward-creation-receipt-charge-responsibility/)
> 3. **AI 대화 후처리를 요청 트랜잭션에서 분리하기** — 현재 글
{: .prompt-info }

---

## 1. 요청에서 꼭 끝나야 하는 작업 구분하기

먼저 사용자에게 응답하기 전에 반드시 끝나야 하는 작업과 나중에 실행해도 되는 작업을 나누어보았습니다.

![요청 안의 작업과 후처리](/assets/images/posts/ai-conversation-post-processing/diagrams/request-vs-post.svg){: .light }
![요청 안의 작업과 후처리](/assets/images/posts/ai-conversation-post-processing/diagrams/request-vs-post-dark.svg){: .dark }
요청 결과는 사용자에게 성공을 반환하기 전에 반드시 저장해야 합니다. 후처리 결과는 나중에 계산할 수 있지만 늦어지거나 실패했을 때 확인할 방법이 필요합니다.

| 질문 | 요청 안에서 처리 | 후처리로 분리 |
|---|---:|---:|
| 없으면 현재 요청이 실패인가? | 예 | 상황에 따라 다름 |
| 즉시 일관성이 필요한가? | 대체로 예 | 별도 검토 |
| 외부 API 지연을 포함하는가? | 가능하면 피함 | 별도 실행 검토 |
| 독립 재시도가 가능한가? | 제한적 | 설계 가능 |

같은 작업이라도 응답에 반드시 필요한지, 다른 데이터가 먼저 저장되어야 하는지에 따라 위치가 달라질 수 있습니다.

---

## 2. 후처리를 요청 밖으로 옮기기

후처리를 요청 밖으로 옮기면 HTTP 응답은 빨라질 수 있습니다. 하지만 실행해야 할 작업과 실패 가능성이 사라지는 것은 아닙니다.

![응답 후 실행되는 후처리](/assets/images/posts/ai-conversation-post-processing/diagrams/async-sequence.svg){: .light }
![응답 후 실행되는 후처리](/assets/images/posts/ai-conversation-post-processing/diagrams/async-sequence-dark.svg){: .dark }
이 구조가 유효하려면 다음 질문에 답해야 합니다.

- 신호가 유실되면 어떻게 발견하는가?
- 같은 신호가 중복 전달되면 안전한가?
- 애플리케이션이 종료될 때 진행 중인 작업을 어떻게 다루는가?
- 후처리 결과는 언제까지 필요한가?
- 운영자가 실패를 재처리할 수 있는가?

“코루틴을 실행했다”는 사실만으로 신뢰할 수 있는 파이프라인이 완성되지는 않습니다. 전달 보장, 멱등성, 관측 가능성과 종료 절차까지 하나의 설계로 봐야 합니다.

---

## 3. 커밋 이후에 작업 시작하기

후처리에서 방금 저장한 데이터를 다시 조회한다면 저장 메서드가 호출된 시점에 바로 시작하면 안 됩니다. 트랜잭션이 정상적으로 커밋된 후에 작업을 시작해야 합니다.

![커밋 결과에 따른 상태](/assets/images/posts/ai-conversation-post-processing/diagrams/commit-states.svg){: .light }
![커밋 결과에 따른 상태](/assets/images/posts/ai-conversation-post-processing/diagrams/commit-states-dark.svg){: .dark }
의사 코드로 표현하면 다음과 같습니다.

~~~kotlin
data class PostProcessRequested(
    val recordId: ExampleRecordId,
    val correlationId: CorrelationId,
)

fun handle(command: ExampleCommand): ExampleResult =
    transaction {
        val saved = saveCoreResult(command)

        afterSuccessfulCommit {
            publish(
                PostProcessRequested(
                    recordId = saved.id,
                    correlationId = command.correlationId,
                )
            )
        }

        saved.toResult()
    }
~~~

<code>afterSuccessfulCommit</code>과 <code>publish</code>는 특정 프레임워크 API가 아니라 개념을 보여주는 가상 함수입니다. 실제 구현은 로컬 이벤트, 트랜잭셔널 아웃박스, 메시지 브로커 등 요구되는 전달 수준에 따라 달라질 수 있습니다.

여기서 두 가지를 확인해야 합니다.

1. 롤백된 데이터에 대한 후처리를 시작하지 않습니다.
2. 커밋과 신호 전달 사이의 실패 가능성을 별도로 다룹니다.

---

## 4. 이벤트 전달 방식 비교하기

그럼 커밋 후 작업을 어떤 방식으로 전달할지 비교해보겠습니다. 프로세스 메모리 안의 이벤트는 구현이 단순하지만 커밋 직후 애플리케이션이 종료되면 신호가 사라질 수 있습니다. 작업 레코드나 메시지 브로커를 사용하면 구성은 복잡해지지만 다시 처리할 근거를 남길 수 있습니다.

| 방식 | 장점 | 반드시 검토할 위험 |
|---|---|---|
| 커밋 후 인메모리 실행 | 단순한 구조, 낮은 도입 비용 | 프로세스 종료 시 유실 가능성 |
| DB 작업 레코드 | 요청 데이터와 작업 예약을 함께 커밋 가능 | 폴링, 정리, 중복 실행 관리 |
| 메시지 브로커 | 소비자 분리와 확장에 유리 | 인프라 운영, 중복 전달, 순서 관리 |

![필요한 전달 보장에 따른 선택](/assets/images/posts/ai-conversation-post-processing/diagrams/delivery-choice.svg){: .light }
![필요한 전달 보장에 따른 선택](/assets/images/posts/ai-conversation-post-processing/diagrams/delivery-choice-dark.svg){: .dark }
어느 방식이 맞는지는 작업의 중요도와 허용 가능한 지연에 따라 달라집니다.

---

## 5. 외부 API와 DB 트랜잭션 나누기

외부 AI API는 응답 시간이 일정하지 않고 타임아웃이나 일시 오류가 발생할 수 있습니다. 이 호출을 DB 트랜잭션 안에서 실행하면 응답을 기다리는 동안 커넥션과 잠금을 계속 사용하게 됩니다.

![외부 API 대기와 트랜잭션 경계](/assets/images/posts/ai-conversation-post-processing/diagrams/transaction-split.svg){: .light }
![외부 API 대기와 트랜잭션 경계](/assets/images/posts/ai-conversation-post-processing/diagrams/transaction-split-dark.svg){: .dark }
외부 호출과 저장은 다음 코드처럼 나눌 수 있습니다.

~~~kotlin
suspend fun runExampleTask(id: ExampleRecordId) {
    val input = loadInputSnapshot(id)
    val generated = externalPort.generate(input)

    shortTransaction {
        saveDerivedResult(
            id = id,
            result = generated,
        )
    }
}
~~~

다만 조회 이후 원본이 변경될 수 있으므로 스냅샷 버전, 입력 해시, 조건부 저장 같은 검증이 필요할 수 있습니다. 어떤 충돌을 허용하고 어떤 결과를 폐기할지는 제품 요구사항에 따라 정해야 합니다.

---

## 6. 후처리 작업 순서 정하기

후처리 작업을 모두 병렬로 실행할 수 있는 것은 아닙니다. 먼저 각 작업이 어떤 결과를 사용하는지 그림으로 정리해보았습니다.

![후처리 작업의 의존 관계](/assets/images/posts/ai-conversation-post-processing/diagrams/task-dependencies.svg){: .light }
![후처리 작업의 의존 관계](/assets/images/posts/ai-conversation-post-processing/diagrams/task-dependencies-dark.svg){: .dark }
| 관계 | 실행 방법 예시 |
|---|---|
| 서로 입력과 출력이 독립 | 제한된 동시성으로 병렬 실행 |
| 앞 작업의 결과가 필요 | 명시적 순차 실행 |
| 일부 실패를 허용 | 실패 격리와 결과 상태 기록 |
| 모두 성공해야 의미 있음 | 묶음 실패 처리 또는 보상 전략 |

호출 순서에 우연히 의존하면 리팩터링 과정에서 규칙이 깨질 수 있습니다. 의존성이 실제 비즈니스 규칙이라면 코드 구조나 오케스트레이션 모델에 명시적으로 드러나야 합니다.

---

## 7. 코루틴 스코프 정하기

코루틴을 사용하기 전에 작업을 어느 스코프에서 실행할지 정해야 합니다. 요청 스코프에서 시작한 작업은 요청이 끝날 때 취소될 수 있고 애플리케이션 스코프에서 실행한 작업은 종료 시점을 직접 관리해야 합니다.

![애플리케이션이 관리하는 스코프](/assets/images/posts/ai-conversation-post-processing/diagrams/coroutine-scope.svg){: .light }
![애플리케이션이 관리하는 스코프](/assets/images/posts/ai-conversation-post-processing/diagrams/coroutine-scope-dark.svg){: .dark }
검토할 항목은 다음과 같습니다.

- 요청 스코프와 애플리케이션 스코프 중 누가 작업을 소유하는가?
- 한 자식 작업의 실패가 형제 작업을 취소해야 하는가?
- 동시 실행 수는 어디에서 제한하는가?
- 블로킹 클라이언트를 어떤 실행 문맥에서 호출하는가?
- 종료 시 새 작업 수락을 멈추고 진행 작업을 기다릴 것인가?
- 제한 시간 이후 작업을 취소하면 재처리 근거가 남는가?

~~~kotlin
// 구조를 보여주기 위한 의사 코드
suspend fun processExample(event: PostProcessRequested) =
    supervisedTaskGroup {
        val a = async { taskA(event) }
        val b = async { taskB(event) }

        taskC(
            first = a.await(),
            second = b.await(),
        )
    }
~~~

<code>supervisedTaskGroup</code>도 개념을 보여주는 가상 함수입니다. 실패 격리가 필요하다는 이유만으로 모든 예외를 삼키면 안 됩니다. 실패는 상태, 메트릭 또는 재처리 가능한 레코드로 관측되어야 합니다.

---

## 8. 중복 실행 막기

이벤트나 작업 신호는 같은 내용이 두 번 전달될 수 있습니다. 그래서 같은 작업인지 확인할 수 있는 키를 함께 저장해야 합니다.

![같은 작업 신호의 중복 처리](/assets/images/posts/ai-conversation-post-processing/diagrams/idempotent-worker.svg){: .light }
![같은 작업 신호의 중복 처리](/assets/images/posts/ai-conversation-post-processing/diagrams/idempotent-worker-dark.svg){: .dark }
멱등성 키는 이벤트 식별자, 원본 레코드 버전, 작업 종류의 조합 등으로 만들 수 있습니다. 재시도할 때 같은 작업인지 판별할 수 있어야 합니다.

~~~sql
CREATE TABLE example_async_operation (
    operation_key   VARCHAR(200) PRIMARY KEY,
    operation_kind  VARCHAR(100) NOT NULL,
    result_state    VARCHAR(100) NOT NULL,
    updated_at      TIMESTAMP NOT NULL
);
~~~

성공 전에 키를 선점할지, 실패 상태를 덮어쓸지, 얼마나 보관할지도 운영 정책에 맞춰 함께 정해야 합니다.

---

## 9. 작업별 실패 처리 정하기

후처리 전체를 하나의 <code>try/catch</code>로 감싸면 어느 작업에서 실패했는지 확인하기 어렵습니다. 그래서 작업마다 재시도 가능 여부와 사용자 영향을 나누어 정리했습니다.

| 항목 | 결정해야 할 질문 |
|---|---|
| 재시도 가능성 | 일시 오류와 영구 오류를 어떻게 구분하는가? |
| 최대 지연 | 결과가 언제까지 생성되어야 하는가? |
| 중복 안전성 | 같은 입력을 다시 처리해도 안전한가? |
| 선행 의존성 | 앞 작업 실패 시 실행할 수 있는가? |
| 사용자 영향 | 결과 지연을 사용자에게 보여줘야 하는가? |
| 운영 대응 | 자동 재처리와 수동 복구의 경계는 어디인가? |

![작업 실패 분류](/assets/images/posts/ai-conversation-post-processing/diagrams/failure-handling.svg){: .light }
![작업 실패 분류](/assets/images/posts/ai-conversation-post-processing/diagrams/failure-handling-dark.svg){: .dark }
재시도 횟수, 간격, 격리 저장 기간 같은 값은 시스템 용량과 작업의 중요도에 맞춰 정해야 합니다.

---

## 10. 후처리 상태 기록하기

사용자가 이미 성공 응답을 받은 뒤에는 후처리 실패가 HTTP 오류로 나타나지 않습니다. 별도의 로그와 메트릭이 없으면 작업이 누락돼도 확인하기 어렵습니다.

![후처리 상태와 관측 신호](/assets/images/posts/ai-conversation-post-processing/diagrams/post-observability.svg){: .light }
![후처리 상태와 관측 신호](/assets/images/posts/ai-conversation-post-processing/diagrams/post-observability-dark.svg){: .dark }
후처리 상태는 다음과 같은 지표로 확인할 수 있습니다.

- 큐 또는 작업 레코드의 대기 시간
- 작업 종류별 성공·실패 비율
- 외부 API 지연과 오류 분류
- 생성 시점부터 최종 완료까지의 종단 지연
- 같은 멱등성 키의 중복 감지 횟수
- 재처리 대기 항목 수

로그에는 원문 대화나 프롬프트 대신 상관관계 식별자와 작업 상태를 남기는 편이 개인정보와 디버깅 사이의 균형을 잡는 데 유리합니다.

---

## 11. 실패와 중복 상황 테스트하기

마지막으로 응답 시간뿐 아니라 작업 유실과 중복 실행도 함께 테스트했습니다.

| 검증 범주 | 확인할 내용 |
|---|---|
| 커밋 경계 | 롤백된 핵심 데이터로 작업이 시작되지 않는다 |
| 외부 지연 | 외부 API 대기 동안 긴 DB 트랜잭션을 유지하지 않는다 |
| 중복 전달 | 같은 작업이 반복되어도 부수효과가 중복되지 않는다 |
| 프로세스 종료 | 종료 전후에 유실·복구 상태를 식별할 수 있다 |
| 부분 실패 | 독립 작업의 실패 범위가 설계와 일치한다 |
| 역압 | 유입량이 처리량을 넘을 때 제한 장치가 작동한다 |
| 관측성 | 실패 항목을 찾고 재처리 근거를 확인할 수 있다 |

부하 테스트에서도 평균 응답 시간뿐 아니라 꼬리 지연, DB 커넥션 점유, 외부 API 동시 요청 수와 작업 적체를 함께 봐야 합니다.

---

## 12. 마무리

1. 요청 결과와 후처리 결과의 완료 조건을 먼저 구분합니다.
2. 후처리는 요청 데이터가 커밋된 뒤 시작합니다.
3. 커밋 후 실행과 내구성 있는 전달을 같은 것으로 보지 않습니다.
4. 외부 API 호출은 긴 DB 트랜잭션 밖에 둡니다.
5. 작업 사이의 의존성과 실패 전파를 명시합니다.
6. 중복 전달을 전제로 멱등성 키와 상태 기록을 설계합니다.
7. 응답 이후의 실패를 발견할 메트릭과 재처리 경로를 준비합니다.

이번 글에서는 AI 요청의 저장과 후처리를 나누고 커밋 시점, 외부 API, 코루틴 스코프, 중복 실행과 실패 기록을 차례로 살펴보았습니다. 코루틴으로 요청을 빠르게 반환하는 것뿐 아니라, 요청 밖으로 옮긴 작업을 다시 확인하고 처리할 수 있게 만드는 과정이 함께 필요했습니다.

---
title: "리워드 생성·수령·충전 책임 분리 개발기"
date: 2026-03-20
category: 개발기
tags: [spring-boot, kotlin, domain-driven-design, reward, transaction, architecture, postgres]
readTime: 14min
mermaid: true
image:
  path: /assets/images/posts/reward-responsibility/building-blocks.jpg
  thumbnail: /assets/images/posts/reward-responsibility/building-blocks.jpg
---

# 리워드 생성·수령·충전 책임 분리 개발기

리워드 기능을 하나의 메서드에서 처리하면 처음에는 구현하기 쉽습니다. 하지만 생성, 상태 변경과 실제 반영이 한곳에 모이면 수정할 때마다 여러 책임을 함께 확인해야 합니다.

이번 글에서는 리워드의 생성·수령·충전 책임을 나누고, 각 단계의 결과를 다음 유스케이스로 전달하는 구조를 정리해보겠습니다.

> 아래 도메인 이름, 상태, 처리 순서, 코드와 스키마는 설명을 위해 만든 예제입니다. 실제 보상 조건, 지급량, 수령 방식과 운영 규칙은 포함하지 않았습니다.
{: .prompt-info }

![분리된 모듈을 상징하는 블록](/assets/images/posts/reward-responsibility/building-blocks.jpg)

<small>서로 다른 책임을 가진 구성 요소를 떠올리게 하는 블록. Photo by [Duc Anh Nguyen on Pexels](https://www.pexels.com/photo/colorful-building-blocks-on-a-table-8289844/).</small>

<div class="notice--info" markdown="1">
**연관 개발기**

1. [Spring Boot와 Kotlin Coroutine을 통한 AI 대화 후처리 개발기](/posts/ai-conversation-post-processing-pipeline/)
2. **리워드 생성·수령·충전 책임 분리 개발기** — 현재 글
3. [Spring Data JPA 상태 조건 UPDATE 개발기](/posts/reward-conditional-update-idempotency/)
</div>

---

## 1. 한 메서드에 모인 책임 확인하기

먼저 기존 흐름에서 한 메서드가 담당하고 있는 일을 나누어보았습니다.

- 어떤 사건을 근거로 권리가 생겼는가?
- 그 권리는 아직 사용할 수 있는가?
- 자산 변화가 정확히 한 번 기록됐는가?

세 질문은 비슷해 보여도 확인할 데이터와 조건이 다릅니다.

| 개념 | 설명을 위한 이름 | 확인할 조건 |
|---|---|---|
| 권리 생성 | Entitlement Creation | 같은 근거로 의도치 않은 권리를 중복 생성하지 않는다 |
| 상태 전이 | Entitlement Transition | 허용된 이전 상태에서만 다음 상태로 이동한다 |
| 효과 반영 | Effect Application | 변경 결과와 근거 기록을 함께 남긴다 |

~~~mermaid
flowchart LR
    S["가상의 사건"] --> C["권리 생성"]
    C --> T["상태 전이"]
    T --> E["효과 반영"]

    C -. "생성 조건" .-> CI["중복 근거 확인"]
    T -. "변경 조건" .-> TI["현재 상태 확인"]
    E -. "반영 조건" .-> EI["원장과 결과 확인"]
~~~

여기서 화살표는 실제 서비스의 처리 순서를 뜻하지 않습니다. 각 책임이 어떤 결과를 다음 책임에 제공할 수 있는지 보여주는 개념도입니다.

---

## 2. 권리와 자산 기록 나누기

권리는 어떤 처리를 실행할 수 있는 근거이고, 자산 기록은 실제로 무엇이 변경됐는지 보여줍니다. 두 개념을 한 레코드에 넣으면 생성 여부와 반영 여부를 구분하기 어려워집니다.

~~~mermaid
classDiagram
    class ExampleEntitlement {
        ExampleId id
        SubjectId subjectId
        EvidenceKey evidenceKey
        ExampleState state
    }

    class ExampleEffect {
        EffectId id
        ExampleId entitlementId
        EffectValue value
    }

    class ExampleLedger {
        LedgerId id
        ReferenceKey referenceKey
        Delta delta
    }

    ExampleEntitlement --> ExampleEffect : 적용 근거
    ExampleEffect --> ExampleLedger : 변화 기록
~~~

이제 각 데이터가 담당하는 내용을 나누어보겠습니다.

1. 권리 레코드는 생성 근거와 생명주기를 설명합니다.
2. 효과 객체는 무엇을 반영할지 표현합니다.
3. 원장은 반영된 변화와 추적 근거를 남깁니다.

이 그림이 물리 테이블 세 개를 반드시 만들어야 한다는 뜻은 아닙니다. 먼저 코드에서 책임을 나누고, 저장 구조는 함께 변경해야 하는 데이터와 조회 비용을 확인한 후 정할 수 있습니다.

---

## 3. 유스케이스 연결하기

서비스를 나눈 후 서로의 저장소를 직접 수정하게 두면 클래스만 늘어나고 결합은 그대로 남습니다. 그래서 앞 단계가 다음 단계에 필요한 결과만 반환하도록 구성하였습니다.

~~~kotlin
// 공개 설명을 위한 가상 타입이다.
data class CreatedEntitlement(
    val id: ExampleId,
)

data class TransitionedEntitlement(
    val id: ExampleId,
    val transitionToken: TransitionToken,
)

interface CreateEntitlement {
    fun execute(command: ExampleCreateCommand): CreatedEntitlement
}

interface TransitionEntitlement {
    fun execute(command: ExampleTransitionCommand): TransitionedEntitlement
}

interface ApplyEffect {
    fun execute(command: ExampleEffectCommand)
}
~~~

여기서 <code>TransitionedEntitlement</code>는 “상태 전이에 성공한 대상”만 다음 책임으로 전달하기 위한 예시입니다. 실제 반환 타입이나 호출 구조를 나타내지 않습니다.

~~~mermaid
flowchart TB
    I["입력"] --> UC1["Create Entitlement"]
    UC1 --> O1["CreatedEntitlement"]
    O1 --> UC2["Transition Entitlement"]
    UC2 --> O2["TransitionedEntitlement"]
    O2 --> UC3["Apply Effect"]
    UC3 --> O3["Effect Result"]

    DB1[("권리 저장소")] --- UC1
    DB1 --- UC2
    DB2[("자산 저장소")] --- UC3
~~~

이 방식은 다음과 같은 실수를 줄여 줍니다.

- 생성 서비스가 자산 저장소의 세부 구현까지 아는 문제
- 상태를 바꾸지 못했는데도 효과 반영을 시도하는 문제
- 호출자가 내부 엔티티를 수정해 생명주기 규칙을 우회하는 문제

---

## 4. 상태 조건 UPDATE 연결하기

이제 수령 단계의 상태 변경을 살펴보겠습니다. 먼저 상태를 읽고 나중에 저장하면 두 요청이 같은 이전 상태를 확인할 수 있습니다.

~~~mermaid
sequenceDiagram
    participant A as 요청 A
    participant B as 요청 B
    participant DB as 저장소

    A->>DB: 상태 조회
    B->>DB: 상태 조회
    DB-->>A: 같은 이전 상태
    DB-->>B: 같은 이전 상태
    A->>DB: 상태 변경
    B->>DB: 상태 변경 시도
~~~

이를 막기 위해 이전 상태를 UPDATE의 WHERE 조건에 포함할 수 있습니다.

~~~sql
UPDATE entitlement_example
SET state = :next_state
WHERE id = :id
  AND state = :expected_state;
~~~

영향받은 행이 한 건이면 전이 성공, 없으면 누군가 먼저 처리했거나 입력 조건이 맞지 않은 것으로 해석할 수 있습니다. 단, “행이 없음”을 어떤 사용자 응답으로 바꿀지는 제품 정책이므로 이 글에서는 다루지 않습니다.

상태 조건 UPDATE의 동시성 원리와 저장 컨텍스트 주의점은 [상태 조건 UPDATE로 중복 수령 방지](/posts/reward-conditional-update-idempotency/)에서 별도로 정리했습니다.

---

## 5. 변경 원장 남기기

값만 변경하면 최종 결과는 확인할 수 있지만 어떤 요청으로 변경됐는지 다시 찾기 어렵습니다. 그래서 변경 결과와 근거가 되는 원장 기록을 함께 다루어야 합니다.

~~~mermaid
flowchart LR
    V["전이에 성공한 가상 권리"] --> L["원장 항목 생성"]
    L --> B["잔액 변화"]
    B --> C["커밋"]
    L -. "함께 처리할 범위" .- B
~~~

가상의 원장 모델은 다음처럼 최소한의 참조 키를 가질 수 있습니다.

~~~sql
CREATE TABLE example_asset_ledger (
    ledger_id       UUID PRIMARY KEY,
    subject_key     UUID NOT NULL,
    reference_key   UUID NOT NULL,
    delta_value     BIGINT NOT NULL,
    recorded_at     TIMESTAMP NOT NULL,
    UNIQUE (reference_key)
);
~~~

이 스키마는 실제 테이블이나 컬럼이 아닙니다. <code>reference_key</code>에 유일성 제약을 두어 같은 근거의 중복 반영을 방어할 수 있다는 설계 예시입니다.

애플리케이션에서 먼저 조회하는 것만으로는 동시 요청을 완전히 막기 어렵습니다. 조건부 상태 변경과 원장 유일성 제약을 함께 사용하면 한쪽의 검사가 누락되어도 중복 기록을 한 번 더 막을 수 있습니다.

---

## 6. 트랜잭션 범위 정하기

책임을 세 개로 나누었다고 트랜잭션도 반드시 세 개가 되는 것은 아닙니다. 실패했을 때 함께 되돌려야 하는 작업을 기준으로 트랜잭션 범위를 정하였습니다.

~~~mermaid
flowchart TB
    subgraph TX["가상의 원자적 처리 범위"]
        T["허용된 상태 전이"]
        L["원장 기록"]
        B["잔액 반영"]
        T --> L --> B
    end

    C["권리 생성"] -. "별도 트랜잭션 가능" .-> TX
~~~

트랜잭션 범위를 정할 때에는 다음 내용을 확인하였습니다.

- 상태는 바뀌었지만 자산은 반영되지 않아도 되는가?
- 원장은 생겼지만 잔액이 바뀌지 않은 상태를 허용하는가?
- 외부 시스템 호출이 있다면 DB 롤백으로 되돌릴 수 있는가?
- 재시도할 때 같은 근거를 식별할 키가 있는가?

함께 성공하거나 함께 실패해야 한다면 하나의 로컬 트랜잭션으로 묶을 수 있습니다. 외부 시스템 호출이 포함된다면 DB 롤백만으로 되돌릴 수 없으므로 아웃박스, 멱등 키나 별도의 보정 작업을 검토해야 합니다.

---

## 7. 저장 Port 나누기

헥사고날 구조에서 포트를 지나치게 세분화하면 이름만 다른 인터페이스가 늘어날 수 있습니다. 반대로 하나의 저장 포트에 모든 기능을 모으면 권리와 자산의 변경 이유가 다시 결합됩니다.

~~~mermaid
flowchart LR
    APP["Application"]
    EP["Entitlement Port"]
    AP["Asset Port"]
    OP["Operation Record Port"]

    APP --> EP
    APP --> AP
    APP --> OP

    EP --> EDB[("Persistence Adapter A")]
    AP --> ADB[("Persistence Adapter B")]
    OP --> ODB[("Persistence Adapter C")]
~~~

포트의 적절한 크기는 팀과 코드베이스에 따라 다릅니다. 다만 다음 기준은 비교적 재사용하기 좋았습니다.

- 같은 조건을 확인하는 연산은 함께 둔다.
- 호출자가 알 필요 없는 물리 저장 구조는 숨긴다.
- 도메인 결과와 저장 엔티티를 무조건 동일시하지 않는다.
- 테스트에서 실패 지점을 독립적으로 표현할 수 있게 한다.

---

## 8. 여러 건의 처리 결과 구분하기

여러 건을 한 번에 처리할 때 변경된 개수만 반환하면 어떤 항목이 실제로 성공했는지 확인하기 어렵습니다. 다음 작업에 정확한 대상을 전달하기 위해 성공한 식별자를 따로 보존해야 합니다.

~~~text
입력 식별자 집합
        │
        ▼
조건부 상태 전이
        │
        ├── 성공 식별자 집합
        └── 전이되지 않은 식별자 집합
~~~

데이터베이스의 <code>RETURNING</code>, 항목별 조건부 UPDATE, 처리 토큰 등 여러 구현 전략이 있습니다. 어떤 전략이 적절한지는 데이터 양, 잠금 비용, 부분 성공 허용 여부와 데이터베이스 기능에 따라 달라집니다.

부분 성공을 허용할지, 전체를 롤백할지, 재시도를 어떻게 보여줄지는 제품 정책입니다. 공개 예시에서는 한 가지를 정답처럼 제시하지 않고 결정이 필요한 지점으로만 남겼습니다.

---

## 9. 동시성과 실패 상황 테스트하기

마지막으로 동시 요청과 중간 실패 상황을 테스트하였습니다.

| 테스트 범주 | 확인할 내용 |
|---|---|
| 생성 중복 | 같은 근거가 의도치 않게 여러 권리를 만들지 않는다 |
| 상태 경합 | 동일한 이전 상태에서 한 전이만 성공한다 |
| 원장 중복 | 같은 참조 키의 자산 변화가 중복 기록되지 않는다 |
| 중간 실패 | 함께 저장해야 하는 결과가 일부만 남지 않는다 |
| 재시도 | 같은 명령이 다시 와도 허용 범위를 넘는 부수효과가 없다 |

~~~kotlin
@Test
fun one_transition_wins_for_the_same_example_id() {
    // Given: 같은 가상 권리를 가리키는 동시 명령
    // When: 여러 실행자가 조건부 전이를 시도
    // Then: 성공한 전이의 수가 정한 조건과 일치
}

@Test
fun the_same_reference_does_not_create_duplicate_effects() {
    // Given: 같은 가상 참조 키
    // When: 효과 반영 명령이 반복
    // Then: 원장과 자산 변화가 중복되지 않음
}
~~~

테스트 이름과 타입 역시 설명용입니다. 실제 정책값 대신 상태 변경 횟수와 중복 저장 여부를 확인하였습니다.

---

## 10. 마무리

이번 작업에서 확인한 내용을 정리하면 다음과 같습니다.

1. 권리 생성, 상태 전이, 자산 반영은 서로 다른 책임으로 모델링할 수 있습니다.
2. 책임 분리와 트랜잭션 분리는 같은 결정이 아닙니다.
3. 상태 전이는 기대 상태를 쓰기 조건에 포함하면 경합을 더 명확하게 다룰 수 있습니다.
4. 원장과 유일성 제약은 중복 부수효과를 막는 중요한 방어선입니다.
5. 여러 건 처리에서는 개수보다 실제 성공한 대상을 추적해야 합니다.
6. 구체적인 지급 조건과 실패 응답은 구조가 아니라 제품 정책이므로 별도로 관리해야 합니다.

이번 글에서는 리워드 생성, 상태 변경과 자산 반영을 각각 나누고 다시 하나의 흐름으로 연결해보았습니다. 책임을 나누는 목적은 클래스를 늘리는 것이 아니라, 각 단계에서 확인해야 할 조건과 실패 범위를 분명하게 만드는 데 있었습니다.

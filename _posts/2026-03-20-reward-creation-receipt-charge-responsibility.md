---
title: "리워드 책임 분리 개발기 — 생성,수령, 충전을 하나의 흐름에서 떼어내기"
date: 2026-03-20
category: 개발기
tags: [spring-boot, kotlin, domain-driven-design, reward, transaction, architecture, postgres]
readTime: 14min
mermaid: true
image:
  path: /assets/images/posts/reward-responsibility/building-blocks.jpg
  thumbnail: /assets/images/posts/reward-responsibility/building-blocks.jpg
---

# 리워드 책임 분리 개발기 — 생성, 수령, 충전을 하나의 흐름에서 떼어내기

보상 기능은 처음에는 단순해 보입니다. 어떤 사건이 발생하면 기록을 만들고, 사용자의 자산을 변경하면 됩니다. 하지만 기능이 커질수록 “권리가 생기는 일”, “권리를 사용한 것으로 바꾸는 일”, “실제 자산을 반영하는 일”은 서로 다른 실패 조건과 동시성 문제를 갖게 됩니다.

이 글은 세 책임을 분리해 생각하는 일반적인 설계 방법을 정리한 개발기입니다. 특정 서비스의 현재 구조를 설명하거나 운영 정책을 공개하는 문서가 아닙니다.

> 이 글의 도메인 이름, 상태, 처리 순서, 코드, 테이블과 필드는 모두 설명을 위한 가상 예시입니다. 실제 보상 발생 조건, 지급량, 상품 종류, 수령 방식, 예외 규칙, API, 스키마, 트랜잭션 경계와 운영 기준은 포함하지 않았습니다.
{: .prompt-info }

![분리된 모듈을 상징하는 블록](/assets/images/posts/reward-responsibility/building-blocks.jpg)

<small>서로 다른 책임을 가진 구성 요소를 떠올리게 하는 블록. Photo by [Duc Anh Nguyen on Pexels](https://www.pexels.com/photo/colorful-building-blocks-on-a-table-8289844/).</small>

<div class="notice--info" markdown="1">
**연관 개발기**

1. [AI 대화 후처리 개선기](/posts/ai-conversation-post-processing-pipeline/)
2. **리워드 생성·수령·충전 책임 분리** — 현재 글
3. [상태 조건 UPDATE로 중복 수령 방지](/posts/reward-conditional-update-idempotency/)
</div>

---

## 1. 메서드를 나누기 전에 의미부터 나눈다

한 메서드가 다음 세 질문에 모두 답하고 있다면 책임 경계가 흐려졌다는 신호일 수 있습니다.

- 어떤 사건을 근거로 권리가 생겼는가?
- 그 권리는 아직 사용할 수 있는가?
- 자산 변화가 정확히 한 번 기록됐는가?

세 질문은 비슷해 보여도 필요한 데이터와 불변식이 다릅니다.

| 개념 | 설명을 위한 이름 | 지키려는 불변식 |
|---|---|---|
| 권리 생성 | Entitlement Creation | 같은 근거로 의도치 않은 권리를 중복 생성하지 않는다 |
| 상태 전이 | Entitlement Transition | 허용된 이전 상태에서만 다음 상태로 이동한다 |
| 효과 반영 | Effect Application | 자산 변화와 근거 기록의 정합성을 유지한다 |

~~~mermaid
flowchart LR
    S["가상의 사건"] --> C["권리 생성"]
    C --> T["상태 전이"]
    T --> E["효과 반영"]

    C -. "생성 불변식" .-> CI["중복 근거 확인"]
    T -. "전이 불변식" .-> TI["현재 상태 확인"]
    E -. "자산 불변식" .-> EI["원장과 잔액 정합성"]
~~~

여기서 화살표는 실제 서비스의 처리 순서를 뜻하지 않습니다. 각 책임이 어떤 결과를 다음 책임에 제공할 수 있는지 보여주는 개념도입니다.

---

## 2. 권리와 자산 기록을 같은 것으로 보지 않는다

권리는 “어떤 효과를 적용할 수 있는 근거”이고, 자산 기록은 “실제로 무엇이 변했는지”를 설명합니다. 둘을 하나의 레코드로 표현하면 생성 여부와 반영 여부가 뒤섞이기 쉽습니다.

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

이 모델에서 중요한 것은 이름이 아니라 분리된 의미입니다.

1. 권리 레코드는 생성 근거와 생명주기를 설명합니다.
2. 효과 객체는 무엇을 반영할지 표현합니다.
3. 원장은 반영된 변화와 추적 근거를 남깁니다.

실제 제품이 어떤 모델을 선택하는지는 요구사항에 따라 달라집니다. 모든 시스템이 세 개의 물리 테이블을 가져야 한다는 뜻도 아닙니다. 논리적 책임을 먼저 나누고, 저장 구조는 정합성과 조회 비용을 고려해 결정하는 편이 안전합니다.

---

## 3. 유스케이스의 출력으로 다음 책임을 연결한다

분리된 서비스가 서로의 내부 저장소를 직접 조작하면 경계만 늘고 결합도는 그대로입니다. 대신 앞 단계가 다음 단계에 필요한 최소 결과를 반환하도록 설계할 수 있습니다.

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

## 4. 상태 전이는 읽기와 쓰기를 하나의 조건으로 표현한다

상태 기반 기능에서 흔한 실수는 먼저 읽고, 애플리케이션에서 판단한 다음, 나중에 갱신하는 것입니다. 동시에 두 요청이 들어오면 둘 다 같은 이전 상태를 읽을 수 있습니다.

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

일반적인 대안은 기대 상태를 UPDATE 조건에 포함하는 것입니다.

~~~sql
UPDATE entitlement_example
SET state = :next_state
WHERE id = :id
  AND state = :expected_state;
~~~

영향받은 행이 한 건이면 전이 성공, 없으면 누군가 먼저 처리했거나 입력 조건이 맞지 않은 것으로 해석할 수 있습니다. 단, “행이 없음”을 어떤 사용자 응답으로 바꿀지는 제품 정책이므로 이 글에서는 다루지 않습니다.

상태 조건 UPDATE의 동시성 원리와 저장 컨텍스트 주의점은 [상태 조건 UPDATE로 중복 수령 방지](/posts/reward-conditional-update-idempotency/)에서 별도로 정리했습니다.

---

## 5. 원장은 부가 로그가 아니라 정합성 경계다

잔액 숫자만 바뀌면 결과는 알 수 있어도 이유를 재구성하기 어렵습니다. 그래서 자산 변화와 근거 기록을 같은 정합성 경계에서 다루는 설계가 자주 사용됩니다.

~~~mermaid
flowchart LR
    V["전이에 성공한 가상 권리"] --> L["원장 항목 생성"]
    L --> B["잔액 변화"]
    B --> C["커밋"]
    L -. "같은 트랜잭션 후보" .- B
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

애플리케이션의 사전 검사만으로는 동시 요청을 완전히 막기 어렵습니다. 조건부 상태 전이와 원장 유일성 제약처럼 서로 다른 계층의 방어를 조합하면 한 계층의 실수가 전체 정합성 훼손으로 이어질 가능성을 낮출 수 있습니다.

---

## 6. 트랜잭션 경계는 실패 시 함께 되돌릴 범위다

책임을 분리했다고 해서 반드시 트랜잭션까지 세 개로 나눌 필요는 없습니다. 객체의 책임과 원자성 경계는 별개의 설계 결정입니다.

~~~mermaid
flowchart TB
    subgraph TX["가상의 원자적 처리 범위"]
        T["허용된 상태 전이"]
        L["원장 기록"]
        B["잔액 반영"]
        T --> L --> B
    end

    C["권리 생성"] -. "별도 경계일 수도 있음" .-> TX
~~~

경계를 정할 때는 다음 질문이 유용했습니다.

- 상태는 바뀌었지만 자산은 반영되지 않아도 되는가?
- 원장은 생겼지만 잔액이 바뀌지 않은 상태를 허용하는가?
- 외부 시스템 호출이 있다면 DB 롤백으로 되돌릴 수 있는가?
- 재시도할 때 같은 근거를 식별할 키가 있는가?

답이 “함께 성공하거나 함께 실패해야 한다”면 하나의 로컬 트랜잭션 후보입니다. 외부 시스템까지 포함한다면 로컬 트랜잭션만으로 해결할 수 없으므로 아웃박스, 멱등 키, 보상 작업 같은 별도의 전달 전략을 검토해야 합니다.

---

## 7. 포트는 기술보다 변경 이유를 기준으로 나눈다

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

- 같은 불변식을 지키는 연산은 함께 둔다.
- 호출자가 알 필요 없는 물리 저장 구조는 숨긴다.
- 도메인 결과와 저장 엔티티를 무조건 동일시하지 않는다.
- 테스트에서 실패 지점을 독립적으로 표현할 수 있게 한다.

---

## 8. 여러 건 처리에서는 “성공한 대상”을 잃지 않는다

여러 권리를 한 번에 처리할 때 단순히 전체 개수만 반환하면 어떤 항목이 실제로 전이에 성공했는지 알기 어렵습니다. 이후 효과 반영 대상을 안전하게 고르려면 성공한 식별자를 보존해야 합니다.

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

## 9. 테스트는 메서드 호출보다 불변식을 검증한다

책임 분리의 효과는 동시 요청과 실패 상황에서 드러납니다.

| 테스트 범주 | 검증할 일반 불변식 |
|---|---|
| 생성 중복 | 같은 근거가 의도치 않게 여러 권리를 만들지 않는다 |
| 상태 경합 | 동일한 이전 상태에서 한 전이만 성공한다 |
| 원장 중복 | 같은 참조 키의 자산 변화가 중복 기록되지 않는다 |
| 중간 실패 | 원장과 잔액이 합의한 원자성 경계를 지킨다 |
| 재시도 | 같은 명령이 다시 와도 허용 범위를 넘는 부수효과가 없다 |

~~~kotlin
@Test
fun one_transition_wins_for_the_same_example_id() {
    // Given: 같은 가상 권리를 가리키는 동시 명령
    // When: 여러 실행자가 조건부 전이를 시도
    // Then: 성공한 전이의 수가 설계한 불변식과 일치
}

@Test
fun the_same_reference_does_not_create_duplicate_effects() {
    // Given: 같은 가상 참조 키
    // When: 효과 반영 명령이 반복
    // Then: 원장과 자산 변화가 중복되지 않음
}
~~~

테스트 이름과 타입 역시 설명용입니다. 실제 정책값 대신 “정확히 한 번 전이”, “중복 부수효과 없음” 같은 구조적 불변식을 검증하는 것이 핵심입니다.

---

## 10. 정리

이번 설계에서 얻은 일반적인 교훈은 다음과 같습니다.

1. 권리 생성, 상태 전이, 자산 반영은 서로 다른 책임으로 모델링할 수 있습니다.
2. 책임 분리와 트랜잭션 분리는 같은 결정이 아닙니다.
3. 상태 전이는 기대 상태를 쓰기 조건에 포함하면 경합을 더 명확하게 다룰 수 있습니다.
4. 원장과 유일성 제약은 중복 부수효과를 막는 중요한 방어선입니다.
5. 여러 건 처리에서는 개수보다 실제 성공한 대상을 추적해야 합니다.
6. 구체적인 지급 조건과 실패 응답은 구조가 아니라 제품 정책이므로 별도로 관리해야 합니다.

좋은 책임 분리는 클래스를 많이 만드는 일이 아니라, 각 변경이 어떤 불변식을 지키는지 분명하게 만드는 일에 가깝습니다. 내부 정책을 코드 곳곳에 흩뜨리지 않고 정책 계층에 모을 수 있다는 점도 이런 분리의 중요한 효과입니다.

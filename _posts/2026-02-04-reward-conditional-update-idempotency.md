---
title: "중복 리워드 수령 방지 개발기 — 조회 후 저장 대신 상태 조건 UPDATE를 선택한 이유"
date: 2026-02-04
category: 개발기
tags: [spring-boot, kotlin, spring-data-jpa, postgres, concurrency, idempotency, reward]
readTime: 16min
mermaid: true
image:
  path: /assets/images/posts/reward-conditional-update/server-rack.jpg
  thumbnail: /assets/images/posts/reward-conditional-update/server-rack.jpg
---

# 중복 리워드 수령 방지 개발기 — 조회 후 저장 대신 상태 조건 UPDATE를 선택한 이유

안녕하세요. duurian 팀에서 백엔드 개발을 담당하고 있는 정지원입니다.

리워드 수령 API의 코드는 처음 보면 단순합니다. 보상을 조회하고, 아직 받지 않았다면 상태를 RECEIVED로 바꾼 뒤 두리안을 충전하면 됩니다. 그러나 모바일 네트워크 재시도, 버튼 연타, 단건 수령과 모두 받기의 동시 호출이 겹치면 “아직 받지 않았다”는 판단을 여러 요청이 동시에 통과할 수 있습니다.

이번 글에서는 애플리케이션의 선조회에 의존하던 판단을 다음 한 문장의 UPDATE로 옮긴 과정을 다룹니다.

> PENDING인 행만 RECEIVED로 바꾸고, 실제로 바뀐 행을 이 요청의 승자로 인정한다.

조건부 UPDATE가 해결한 단건 경쟁, Spring Data JPA에서 <code>@Modifying(clearAutomatically = true)</code>를 사용한 이유, 일괄 수령의 부분 성공 정책을 차례로 살펴봅니다. 마지막에는 현재 구현을 검증하면서 찾은 “부분 경합 후 재조회”의 남은 틈도 함께 다룹니다.

> 코드의 상태 전이 구조는 유지하되, 물리 테이블·컬럼·인덱스·관측 지표 이름과 지급량은 공개용 예시로 일반화했습니다.
{: .prompt-info }

![데이터베이스 상태 전이를 상징하는 서버 랙](/assets/images/posts/reward-conditional-update/server-rack.jpg)

<small>동시 요청의 최종 판정자는 데이터베이스다. Photo by [panumas nikhomkhai on Pexels](https://www.pexels.com/photo/computer-server-in-data-center-room-17489163/).</small>

<div class="notice--info" markdown="1">
**AI 대화와 보상 파이프라인 개발기**

1. [AI 대화 후처리 개선기](/posts/ai-conversation-post-processing-pipeline/)
2. [리워드 생성·수령·충전 책임 분리](/posts/reward-creation-receipt-charge-responsibility/)
3. **상태 조건 UPDATE로 중복 수령 방지** — 현재 글
</div>

---

## 1. 한눈에 보는 선택

| 항목 | 기존 접근 | 적용한 접근 |
|---|---|---|
| 수령 가능 판정 | SELECT 결과를 애플리케이션에서 확인 | UPDATE의 WHERE 조건으로 판정 |
| 동시 요청의 승자 | 여러 요청이 될 수 있음 | 실제 UPDATE에 성공한 요청 |
| 성공 확인 | 메서드가 예외 없이 끝남 | affected rows가 1 이상 |
| 엔티티 상태 반영 | Dirty Checking | 벌크 JPQL 직접 UPDATE |
| 영속성 컨텍스트 | 기존 엔티티가 남을 수 있음 | UPDATE 뒤 자동 clear |
| 락 전략 | 별도 잠금 없음 | 행 갱신 과정의 DB 동시성 제어 활용 |

~~~mermaid
flowchart LR
    A["수령 요청"] --> B["보상 행 조건부 UPDATE"]
    B --> C{"status = PENDING<br/>조건을 만족했는가?"}
    C -->|예| W["affectedRows = 1<br/>이 요청이 승자"]
    C -->|아니오| L["affectedRows = 0<br/>이미 처리됨"]
    W --> D["두리안 충전"]

    style W fill:#c8e6c9,stroke:#2e7d32,color:#111
    style L fill:#ffcdd2,stroke:#c62828,color:#111
~~~

---

## 2. 왜 SELECT 후 UPDATE만으로는 부족한가

초기 형태를 단순화하면 다음과 같습니다.

~~~kotlin
@Transactional
fun receive(rewardId: UUID) {
    val reward = rewardRepository.findById(rewardId)

    if (reward.status == RewardStatus.RECEIVED) {
        throw AlreadyReceivedRewardException()
    }

    reward.receive()
    chargeDurian(reward)
}
~~~

한 요청만 실행될 때는 문제가 없습니다. 경쟁은 두 요청이 같은 PENDING 상태를 읽을 때 발생합니다.

~~~mermaid
sequenceDiagram
    autonumber
    participant A as 요청 A
    participant B as 요청 B
    participant DB as Reward DB
    participant C as Charge

    par 거의 동시에 조회
        A->>DB: SELECT status
        DB-->>A: PENDING
    and
        B->>DB: SELECT status
        DB-->>B: PENDING
    end

    A->>A: 수령 가능
    B->>B: 수령 가능
    A->>DB: status = RECEIVED
    B->>DB: status = RECEIVED
    A->>C: 보상 금액 충전
    B->>C: 보상 금액 충전
~~~

두 요청 모두 자신의 메모리 안에서는 올바르게 검사했습니다. 문제는 검사와 변경이 하나의 원자적 연산이 아니라는 점입니다.

~~~text
SELECT: 아직 PENDING인가?
        ↓ 다른 요청이 끼어들 수 있는 간격
UPDATE: RECEIVED로 변경
~~~

이를 Check-Then-Act 경쟁이라고 볼 수 있습니다. <code>@Transactional</code>은 각 요청 내부의 원자성을 제공하지만, 다른 트랜잭션이 같은 값을 읽지 못하게 자동으로 직렬화하지는 않습니다.

---

## 3. 상태 머신에서 허용된 간선 하나만 UPDATE한다

Reward의 수령 상태는 한 방향으로만 이동합니다.

~~~mermaid
stateDiagram-v2
    [*] --> PENDING: 보상 생성
    PENDING --> RECEIVED: 수령 성공
    RECEIVED --> RECEIVED: 재요청은 변경 없음
~~~

허용된 상태 전이가 PENDING → RECEIVED 하나뿐이라면, 현재 상태 조회와 변경을 한 SQL에 넣을 수 있습니다.

~~~sql
UPDATE claim_record
SET status = 'RECEIVED',
    changed_at = :changedAt
WHERE id = :claimId
  AND status = 'PENDING';
~~~

이 SQL은 세 가지 일을 동시에 수행합니다.

1. 대상 Reward를 찾습니다.
2. 현재 상태가 PENDING인지 검사합니다.
3. 조건을 만족한 행만 RECEIVED로 바꿉니다.

PostgreSQL UPDATE는 완료 시 갱신한 행 수를 반환합니다. 1이면 이 요청이 상태를 바꿨고, 0이면 조건을 만족하는 행이 없었다는 뜻입니다.

<div class="notice--success" markdown="1">
**비즈니스 결과로 바뀐 affected rows**

행 수는 단순한 DB 메타데이터가 아닙니다.

- 1: 수령권 획득, 충전 진행
- 0: 이미 다른 요청이 수령, 충전 금지
</div>

---

## 4. Spring Data JPA에 조건부 UPDATE 구현하기

현재 <code>RewardJpaRepository</code>는 JPQL 수정 쿼리로 상태 전이를 구현합니다.

~~~kotlin
@Modifying(clearAutomatically = true)
@Query("""
    UPDATE RewardEntity r
    SET r.status = :receivedStatus,
        r.baseEntity.updatedAt = :updatedAt
    WHERE r.id = :id
      AND r.status = :pendingStatus
""")
fun updateStatusToReceived(
    id: UUID,
    receivedStatus: RewardStatus,
    pendingStatus: RewardStatus,
    updatedAt: LocalDateTime,
): Int
~~~

어댑터는 도메인 포트에 상태와 현재 시간을 숨깁니다.

~~~kotlin
override fun updateStatusToReceived(id: UUID): Int {
    val now = LocalDateTime.now(ZoneId.of("Asia/Seoul"))

    return rewardJpaRepository.updateStatusToReceived(
        id = id,
        receivedStatus = RewardStatus.RECEIVED,
        pendingStatus = RewardStatus.PENDING,
        updatedAt = now
    )
}
~~~

서비스는 저장 구현을 모른 채 성공 행 수만 해석합니다.

~~~kotlin
val updatedCount =
    commandRewardPort.updateStatusToReceived(reward.id)

if (updatedCount == 0) {
    throw AlreadyReceivedRewardException()
}

chargeDurianUseCase.chargeDurianByReward(
    ChargeDurianByRewardCommand(
        userId = command.userId,
        rewardId = reward.id,
        productId = reward.durianProductId,
        title = resolveTitleByRewardType(reward.type)
    )
)
~~~

~~~mermaid
flowchart LR
    S["ReceiveRewardService"] --> P["CommandRewardPort"]
    P --> A["CommandRewardPersistenceAdapter"]
    A --> J["RewardJpaRepository"]
    J --> DB["PostgreSQL"]
    DB --> N["affectedRows"]
    N --> S
~~~

---

## 5. 왜 Dirty Checking 대신 직접 UPDATE였는가

JPA 엔티티를 조회해 상태를 바꾸는 방식 자체가 나쁜 것은 아닙니다. 문제는 이번 요구사항의 성공 조건이 “내가 읽은 엔티티가 PENDING이었다”가 아니라 “DB에서 PENDING → RECEIVED 전이를 실제로 수행했다”라는 점입니다.

| 비교 | Dirty Checking | 조건부 직접 UPDATE |
|---|---|---|
| 흐름 | SELECT → 엔티티 변경 → flush | 조건을 포함한 UPDATE |
| 상태 판정 위치 | 애플리케이션 메모리 | 데이터베이스 |
| 성공 결과 | 보통 flush 성공 여부 | 갱신 행 수 |
| 동시성 제어 | version 또는 잠금 별도 필요 | WHERE 상태 조건이 게이트 |
| 엔티티 콜백 | 적용 가능 | 벌크 UPDATE라 우회 |
| 영속성 컨텍스트 | 자연스럽게 추적 | 직접 정리 필요 |

이번 선택은 “직접 UPDATE가 항상 더 빠르다”는 일반론 때문이 아닙니다. 상태 전이의 조건과 성공 판정을 하나의 DB 명령에 묶기 위해서였습니다.

~~~mermaid
flowchart TD
    Q{"이번 변경의 핵심은?"}
    Q -->|엔티티 행위·콜백| D["Dirty Checking 검토"]
    Q -->|조건부 상태 전이·행 수 판정| U["Direct UPDATE 적합"]
    U --> C["영속성 컨텍스트 정리 필요"]
~~~

---

## 6. clearAutomatically가 필요한 이유

JPQL 벌크 UPDATE는 현재 영속성 컨텍스트에 로드된 <code>RewardEntity</code>를 하나씩 수정하지 않고 DB에 직접 명령을 보냅니다.

~~~mermaid
flowchart LR
    subgraph PC["Persistence Context"]
        E1["Reward<br/>status=PENDING"]
    end

    U["JPQL UPDATE"] --> DB["DB<br/>status=RECEIVED"]
    DB -. "자동 반영되지 않음" .-> E1
~~~

같은 트랜잭션 앞부분에서 Reward를 조회했다면 메모리에는 PENDING 엔티티가 남을 수 있습니다. 뒤에서 다시 조회해도 1차 캐시가 오래된 객체를 반환할 위험이 있습니다.

<code>@Modifying(clearAutomatically = true)</code>는 수정 쿼리 실행 뒤 EntityManager를 비웁니다.

~~~mermaid
sequenceDiagram
    participant S as Service
    participant PC as Persistence Context
    participant DB as PostgreSQL

    S->>PC: Reward 조회
    PC->>DB: SELECT
    DB-->>PC: PENDING
    S->>DB: 조건부 UPDATE
    DB-->>S: affectedRows = 1
    S->>PC: clearAutomatically
    S->>PC: Reward 재조회
    PC->>DB: SELECT
    DB-->>S: RECEIVED + DB updatedAt
~~~

Spring Data JPA 공식 문서도 modifying query 뒤 EntityManager에 오래된 엔티티가 남을 수 있으며, 필요할 때 <code>clearAutomatically</code>로 자동 정리할 수 있다고 설명합니다.

<div class="notice--warning" markdown="1">
**clear의 비용도 있다**

영속성 컨텍스트를 비우면 아직 flush하지 않은 다른 변경도 관리 대상에서 빠질 수 있습니다. 현재 수령 흐름처럼 수정 쿼리 앞의 엔티티 변경이 없는 짧은 트랜잭션에서는 의도가 분명하지만, 큰 서비스 메서드에 무심코 붙이면 안 됩니다.
</div>

---

## 7. 두 동시 요청에서 한 요청만 이기는 이유

PostgreSQL의 기본 Read Committed 격리 수준에서 두 UPDATE가 같은 행을 겨루면 뒤 요청은 앞 요청의 종료를 기다립니다. 앞 요청이 커밋하면 뒤 요청은 갱신된 행에 WHERE 조건을 다시 평가합니다.

~~~mermaid
sequenceDiagram
    autonumber
    participant A as 요청 A
    participant B as 요청 B
    participant DB as PostgreSQL
    participant C as Charge

    par 동시에 실행
        A->>DB: UPDATE WHERE status=PENDING
        B->>DB: UPDATE WHERE status=PENDING
    end

    DB-->>A: affectedRows=1
    A->>C: 충전
    A->>DB: COMMIT

    Note over B,DB: B는 행 잠금 대기 후 최신 행에 WHERE 재평가
    DB-->>B: status=RECEIVED이므로 affectedRows=0
    B-->>B: AlreadyReceivedRewardException
~~~

이 패턴을 “낙관적 멱등성”이라고 부를 수 있지만 JPA의 <code>@Version</code> 기반 낙관적 잠금과 같은 구현은 아닙니다.

| 용어 | 여기서의 의미 |
|---|---|
| 낙관적 | 대부분 경쟁하지 않는다고 보고 먼저 잠그지 않음 |
| 상태 조건 | PENDING인 경우에만 변경 |
| 멱등성 효과 | 반복 요청이 추가 충전을 만들지 않도록 게이트 |
| 충돌 판정 | 예외가 아니라 affected rows 0 |

---

## 8. 수령과 충전을 같은 트랜잭션에 둔 이유

상태 전이만 성공하고 충전이 실패하면 사용자는 RECEIVED 상태인데 잔액은 받지 못합니다. 반대로 충전만 커밋되고 상태가 PENDING으로 남으면 재요청에서 다시 충전될 수 있습니다.

~~~mermaid
flowchart LR
    subgraph TX["ReceiveReward 트랜잭션"]
        U["조건부 UPDATE"] --> T["충전 원장 저장"]
        T --> B["잔액 저장"]
    end

    B --> X{모두 성공?}
    X -->|예| C["COMMIT"]
    X -->|아니오| R["전체 ROLLBACK"]
~~~

<code>ReceiveRewardService</code>의 바깥 트랜잭션과 다른 Spring 빈인 <code>ChargeDurianService</code>의 기본 REQUIRED 트랜잭션이 합쳐져 다음 불변식을 만듭니다.

~~~text
Reward = RECEIVED
    ⇔
해당 수령 트랜잭션의 원장과 잔액 변경이 함께 COMMIT
~~~

| 장애 | 최종 Reward | 최종 잔액 | 재시도 |
|---|---|---|---|
| UPDATE 전 장애 | PENDING | 변화 없음 | 가능 |
| UPDATE 후 충전 실패 | 롤백되어 PENDING | 변화 없음 | 가능 |
| 잔액 저장 충돌 | 롤백되어 PENDING | 변화 없음 | 가능 |
| 정상 커밋 | RECEIVED | 증가 | 이후 요청은 0행 |

---

## 9. 일괄 수령: 상태 조건을 IN 절로 확장했다

일괄 수령도 같은 원리를 사용합니다.

~~~kotlin
@Modifying(clearAutomatically = true)
@Query("""
    UPDATE RewardEntity r
    SET r.status = :receivedStatus,
        r.baseEntity.updatedAt = :updatedAt
    WHERE r.id IN :ids
      AND r.status = :pendingStatus
""")
fun bulkUpdateStatusToReceived(
    ids: List<UUID>,
    receivedStatus: RewardStatus,
    pendingStatus: RewardStatus,
    updatedAt: LocalDateTime,
): Int
~~~

예를 들어 요청 시점에 세 보상을 읽었지만 한 건이 먼저 처리됐다면 UPDATE 결과는 2가 될 수 있습니다.

~~~mermaid
flowchart LR
    I["요청 ID<br/>R1, R2, R3"] --> S{"DB 현재 상태"}
    S --> R1["R1 RECEIVED"]
    S --> R2["R2 PENDING"]
    S --> R3["R3 PENDING"]
    R1 --> X["변경 없음"]
    R2 --> O["RECEIVED"]
    R3 --> O
    O --> C["affectedRows = 2"]
~~~

현재 응답 모델은 요청 개수와 수령 개수를 비교해 부분 성공 여부를 데이터로 표현합니다.

~~~kotlin
data class ReceiveRewardsResult(
    val requestedCount: Int = 0,
    val receivedCount: Int,
    val receiveTotalQuantity: Int,
    val results: List<ReceivedRewardInfo>,
    val userDurianBalance: GetUserDurianBalanceResult,
) {
    val hasPartialFailure: Boolean
        get() = receivedCount < requestedCount
}
~~~

단건과 일괄 수령은 같은 충돌을 다르게 표현합니다.

| API | 경쟁에서 0행 | 일부 행만 성공 |
|---|---|---|
| 단건 수령 | AlreadyReceivedRewardException | 해당 없음 |
| 일괄 수령 | 빈 결과 | 결과 데이터에 일부 성공 표시 |

일괄 작업에서 한 건의 선처리 때문에 전체 요청을 실패시키는 대신, 실제 성공 범위를 응답으로 돌려주겠다는 제품 정책입니다.

---

## 10. 코드 검증 중 발견한 남은 틈: 성공 행을 행 수만으로는 식별할 수 없다

조건부 일괄 UPDATE의 반환값은 “몇 행을 바꿨는지”만 알려줍니다. “어떤 ID를 바꿨는지”는 알려주지 않습니다.

현재 서비스는 UPDATE 뒤 요청했던 ID 중 RECEIVED인 보상을 다시 조회합니다.

~~~kotlin
val updatedCount =
    commandRewardPort.bulkUpdateStatusToReceived(rewardIds)

val successfulRewards =
    queryRewardPort.findAllByIdInAndStatus(
        ids = rewardIds,
        status = RewardStatus.RECEIVED
    )

val chargeCommand = ChargeDurianBulkCommand(
    userId = command.userId,
    items = successfulRewards.map { reward ->
        ChargeDurianBulkItem(
            durianType = DurianType.REWARD,
            sourceType = LedgerSourceType.REWARD,
            sourceId = reward.id,
            productId = reward.durianProductId,
            title = resolveTitleByRewardType(reward.type)
        )
    }
)

chargeDurianUseCase.chargeDurianBulk(chargeCommand)
~~~

주석의 의도는 “이번 UPDATE로 성공한 행만 충전”이지만, 부분 경합에서는 이미 다른 트랜잭션이 RECEIVED로 만든 행도 재조회 결과에 포함될 수 있습니다.

### 단건과 일괄 수령이 겹치는 타임라인

~~~mermaid
sequenceDiagram
    autonumber
    participant A as 단건 요청 A
    participant B as 일괄 요청 B
    participant DB as PostgreSQL
    participant C as Charge

    B->>DB: PENDING 조회 → R1, R2
    A->>DB: R1 조건부 UPDATE
    DB-->>A: 1행
    A->>C: R1 충전
    A->>DB: COMMIT

    B->>DB: R1,R2 조건부 UPDATE
    DB-->>B: R2만 성공, affectedRows=1
    B->>DB: R1,R2 중 RECEIVED 재조회
    DB-->>B: R1과 R2 모두 반환
    B->>C: R1과 R2 충전
    Note over C: R1 중복 충전 가능
~~~

<div class="notice--danger" markdown="1">
**보장 범위를 정확히 말하면**

- 단건 조건부 UPDATE는 같은 Reward의 중복 수령을 차단합니다.
- 일괄 UPDATE도 상태 전이 자체는 경쟁에서 안전합니다.
- 그러나 현재의 “RECEIVED 전체 재조회 → 일괄 충전”은 부분 경합 시 이번 요청의 승자 집합을 정확히 식별하지 못합니다.
</div>

조건부 UPDATE 패턴이 잘못된 것이 아니라, 행 수만 반환하는 JPQL과 후속 재조회 사이에 정보가 부족한 것입니다.

---

## 11. 부분 경합까지 닫는 네 가지 방법

### 방법 A. PostgreSQL UPDATE RETURNING 사용

PostgreSQL은 실제로 갱신한 행을 <code>RETURNING</code>으로 돌려줄 수 있습니다.

~~~sql
UPDATE claim_record
SET status = 'RECEIVED',
    changed_at = :changedAt
WHERE id = ANY(:ids)
  AND status = 'PENDING'
RETURNING id, subject_id, reward_kind, item_id, changed_at;
~~~

~~~mermaid
flowchart LR
    U["조건부 UPDATE"] --> R["RETURNING<br/>이번에 바뀐 행"]
    R --> C["그 행만 충전"]
~~~

장점은 승자 집합이 정확하고 왕복 조회도 줄어든다는 점입니다. 단점은 JPQL이 아닌 PostgreSQL 네이티브 쿼리와 결과 매핑이 필요해 DB 종속성이 커집니다.

### 방법 B. ID별 단건 조건부 UPDATE

각 ID에 단건 UPDATE를 실행해 1을 반환한 ID만 모읍니다.

~~~kotlin
val wonIds = rewardIds.filter { rewardId ->
    commandRewardPort.updateStatusToReceived(rewardId) == 1
}
~~~

구현은 단순하고 승자를 정확히 알 수 있지만 N번 UPDATE가 발생합니다. 수령 가능한 보상 수가 작고 상한이 명확하다면 현실적인 선택입니다.

### 방법 C. claimToken을 함께 기록

요청마다 UUID 토큰을 만들고 상태 전이와 함께 저장한 뒤 그 토큰으로 다시 조회합니다.

~~~sql
UPDATE claim_record
SET status = 'RECEIVED',
    operation_token = :requestToken
WHERE id = ANY(:ids)
  AND status = 'PENDING';

SELECT *
FROM claim_record
WHERE operation_token = :requestToken;
~~~

JPQL 벌크 UPDATE를 유지하면서 승자 집합을 식별할 수 있지만 스키마와 도메인에 처리 토큰이 추가됩니다.

### 방법 D. 원장 유일 제약을 2차 방어선으로 둔다

~~~sql
CREATE UNIQUE INDEX uq_asset_ledger_source
ON asset_ledger (
    source_type,
    source_id,
    entry_type
)
WHERE source_type = 'REWARD'
  AND entry_type = 'CREDIT'
  AND deleted = false;
~~~

잘못된 승자 집합이 충전 서비스까지 넘어와도 같은 rewardId의 원장 생성을 DB가 거부합니다. 다만 벌크 중 한 건의 유일성 충돌이 전체 트랜잭션을 롤백할 수 있으므로 충전 커맨드 구성과 오류 정책을 함께 설계해야 합니다.

| 대안 | 승자 식별 | DB 왕복 | 스키마 변경 | DB 종속 | 추천 용도 |
|---|---:|---:|---:|---:|---|
| UPDATE RETURNING | 정확 | 적음 | 없음 | 높음 | PostgreSQL 고정 환경 |
| ID별 단건 UPDATE | 정확 | 많음 | 없음 | 낮음 | 보상 수가 적음 |
| claimToken | 정확 | 보통 | 필요 | 낮음 | 처리 추적도 필요 |
| 원장 UNIQUE | 2차 방어 | 동일 | 필요 | 보통 | 모든 방법과 병행 |

현재 PostgreSQL을 사용하고 한 번에 실제 갱신한 행이 필요한 요구사항이라면 A를 우선 검토하고, D를 방어선으로 추가하는 조합이 가장 명확합니다.

---

## 12. 비관적 락과 Redis 락을 선택하지 않은 이유

조건부 UPDATE 외에도 여러 선택지가 있었습니다.

~~~mermaid
flowchart TD
    Q["중복 수령 제어"] --> P["SELECT FOR UPDATE"]
    Q --> O["JPA @Version"]
    Q --> R["Redis 분산 락"]
    Q --> U["상태 조건 UPDATE"]
~~~

| 방식 | 장점 | 비용·한계 | 이번 요구와의 적합성 |
|---|---|---|---|
| SELECT FOR UPDATE | 이해하기 쉽고 강한 직렬화 | 먼저 SELECT, 락 대기·데드락 고려 | 보상 단건에는 다소 무거움 |
| JPA @Version | 엔티티 모델과 자연스럽게 결합 | 충돌 예외와 재시도 정책 필요 | Reward에 version 추가 필요 |
| Redis 락 | 여러 자원·긴 작업 조율 가능 | 락 만료, 소유권, 장애 복구 복잡 | 단일 DB 행 전이에 과함 |
| 조건부 UPDATE | 한 명령, 행 수로 승패 판정 | 벌크 승자 ID는 별도 해결 | 단방향 상태 전이에 적합 |

이번 문제의 경합 대상은 하나의 PostgreSQL Reward 행이고, 허용 전이는 PENDING → RECEIVED 한 방향입니다. 별도 락 인프라보다 DB가 이미 가진 행 갱신 규칙을 사용하는 편이 단순했습니다.

<div class="notice--info" markdown="1">
**선택 기준**

조건부 UPDATE는 복잡한 임계 구역 전체를 보호하는 범용 락이 아닙니다. “현재 값이 X일 때 Y로 한 번만 바꾼다”는 상태 전이에 특히 잘 맞습니다.
</div>

---

## 13. 동시성 테스트는 시간 지연이 아니라 결과 불변식을 검증해야 한다

현재 저장소에는 Reward 수령 경쟁을 검증하는 통합 테스트가 보이지 않습니다. 이 패턴의 신뢰성을 높이려면 실제 PostgreSQL을 사용하는 Testcontainers 테스트가 필요합니다.

### 단건 경쟁 테스트

~~~kotlin
@Test
fun sameRewardCanBeClaimedOnlyOnce() = runBlocking {
    val rewardId = givenPendingReward()
    val gate = CompletableDeferred<Unit>()

    val results = (1..20).map {
        async(Dispatchers.IO) {
            gate.await()
            runCatching { receiveReward(rewardId) }
        }
    }

    gate.complete(Unit)
    val completed = results.awaitAll()

    assertThat(completed.count { it.isSuccess }).isEqualTo(1)
    assertThat(chargeLedgerCount(rewardId)).isEqualTo(1)
    assertThat(rewardStatus(rewardId)).isEqualTo(RECEIVED)
}
~~~

### 단건 + 일괄 부분 경합 테스트

~~~mermaid
flowchart LR
    A["R1 단건 수령"] --> G["동시 시작 Gate"]
    B["R1,R2 모두 수령"] --> G
    G --> X["두 요청 완료"]
    X --> I1["R1 원장 = 1건"]
    X --> I2["R2 원장 = 1건"]
    X --> I3["잔액 증가 = R1 + R2 정확히 한 번"]
~~~

검증해야 할 것은 어떤 요청이 이겼는지가 아닙니다.

| 불변식 | 기대 |
|---|---|
| Reward 최종 상태 | 대상 모두 RECEIVED |
| Reward별 CHARGE 원장 수 | 정확히 1 |
| 최종 잔액 증가량 | 고유 Reward 금액의 합 |
| 전체 성공 요청 수 | 스케줄링에 따라 달라도 됨 |
| 중복 충전 | 0 |

<code>delay</code>로 우연히 충돌을 만드는 테스트보다 동시 시작 배리어와 반복 실행을 사용해야 재현성이 높습니다.

---

## 14. 운영에서 볼 지표

조건부 UPDATE는 충돌을 오류 로그 하나로만 남기기보다 정상적인 경합 지표로 관찰할 수 있게 해줍니다.

| 메트릭 | 의미 | 이상 신호 |
|---|---|---|
| claim.transition.updated | 실제 상태 전이 수 | 보상 생성 대비 지나치게 낮음 |
| claim.transition.conflict | affected rows 0 | 특정 앱 버전에서 급증 |
| claim.transition.partial | 일괄 일부 성공 | 단건/일괄 UI 경쟁 또는 재시도 |
| claim.credit.ledger | 보상 원장 수 | received와 불일치 |
| claim.credit.amount | 지급 총량 | 캠페인 기준치 초과 |
| optimistic.lock.failure | 잔액 version 충돌 | 동일 사용자 병렬 충전 증가 |

~~~mermaid
flowchart LR
    U["UPDATE 성공 수"] --> C{"CHARGE 원장 수와 같은가?"}
    C -->|예| H["정상"]
    C -->|아니오| A["정합성 경보"]
    R["Reward RECEIVED 수"] --> C
~~~

특히 “RECEIVED 보상 수”와 “보상 충전 원장 수”를 sourceId 기준으로 대조하면 상태는 수령됐지만 충전되지 않았거나, 상태 전이보다 원장이 많은 이상을 찾을 수 있습니다.

---

## 15. 배운 점

### 15.1 트랜잭션은 경쟁을 자동으로 없애지 않는다

두 요청이 각각 트랜잭션 안에서 올바르게 동작해도 같은 PENDING 값을 읽을 수 있습니다. 격리 수준과 실제 SQL을 함께 봐야 합니다.

### 15.2 판정을 데이터가 있는 곳으로 옮겨야 한다

애플리케이션의 if 문보다 DB의 WHERE 절이 현재 상태에 더 가깝습니다. 상태 확인과 변경을 한 명령에 넣으면 경쟁 창이 사라집니다.

### 15.3 affected rows는 도메인 결과가 될 수 있다

0과 1은 단순 실행 통계가 아니라 수령권 경쟁의 패자와 승자를 의미합니다. 이 값을 포트의 반환 타입으로 드러낸 것이 서비스 로직을 단순하게 만들었습니다.

### 15.4 벌크에서는 개수와 식별자가 다르다

“두 건 성공했다”는 정보만으로 어떤 두 건인지 알 수 없습니다. 후속 부수효과가 행별로 달라진다면 UPDATE 결과에서 승자 식별자를 보존해야 합니다.

### 15.5 한 겹의 멱등성에 모든 것을 맡기지 않는다

상태 조건 UPDATE, 거래 원장 유일 제약, 잔액 version, 업무 키 유일 제약은 서로 다른 실패를 막습니다. 재화 시스템은 여러 방어선을 갖는 편이 안전합니다.

---

## 마무리

중복 수령 문제의 핵심은 요청이 두 번 들어오는 것 자체가 아닙니다. 분산 환경에서 요청은 언제든 반복될 수 있습니다. 중요한 것은 반복된 요청 중 누가 실제 상태 전이에 성공했는지 데이터베이스가 판정하고, 승자만 부수효과를 시작하게 만드는 것입니다.

PENDING 조건 UPDATE와 affected rows 판정으로 단건 수령의 경쟁 창을 닫았고, 벌크 UPDATE로 같은 원리를 일괄 처리에 확장했습니다. <code>clearAutomatically</code>로 직접 UPDATE 뒤의 영속성 컨텍스트 문제도 정리했습니다.

동시에 검증 과정에서 “일괄 UPDATE 뒤 RECEIVED 전체 재조회”가 부분 경합의 승자 집합을 정확히 보존하지 못한다는 한계도 확인했습니다. 좋은 개발기는 적용한 패턴의 장점만 나열하는 글이 아니라, 그 패턴이 어디까지 보장하고 어디서 추가 설계가 필요한지 기록하는 글이라고 생각합니다.

---

## 참고 자료

- [Spring Data JPA — Modifying Queries](https://docs.spring.io/spring-data/jpa/reference/3.5/jpa/query-methods.html#jpa.modifying-queries)
- [Spring Data JPA — Modifying API](https://docs.spring.io/spring-data/jpa/reference/api/java/org/springframework/data/jpa/repository/Modifying.html)
- [PostgreSQL — UPDATE](https://www.postgresql.org/docs/current/sql-update.html)
- [PostgreSQL — Transaction Isolation](https://www.postgresql.org/docs/16/transaction-iso.html)
- [Pexels 이미지 원문 — panumas nikhomkhai](https://www.pexels.com/photo/computer-server-in-data-center-room-17489163/)

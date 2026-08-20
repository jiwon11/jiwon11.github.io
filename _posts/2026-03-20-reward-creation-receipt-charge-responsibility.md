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

안녕하세요. duurian 팀에서 백엔드 개발을 담당하고 있는 정지원입니다.

처음 조건 달성형 보상을 만들 때의 흐름은 직관적이었습니다. 조건을 만족하면 리워드를 생성하고 곧바로 두리안을 충전했습니다. 작은 기능일 때는 한 메서드에서 모든 일이 끝나는 편이 이해하기 쉬웠습니다.

문제는 리워드가 늘면서 시작됐습니다. 여러 조건형·이벤트형 보상으로 생성 계기가 다양해졌고, 사용자가 직접 “받기”를 누르는 UI도 추가됐습니다. 생성 시점과 수령 시점 모두에서 충전이 가능해지자 같은 보상이 서로 다른 경로를 통해 두 번 충전되는 사고가 발생했습니다.

이 글은 이 문제를 단순한 중복 호출 버그로 보지 않고, 리워드의 생성·수령·재화 충전 책임을 다시 정의한 과정을 다룹니다.

> 보상 트리거, 구간, 지급량, 타입 이름과 원장 필드는 공개용 예시로 일반화했습니다. 수령형과 즉시 지급형을 분리한 설계 원칙만 실제 구조와 동일합니다.
{: .prompt-info }

![분리된 모듈을 상징하는 블록](/assets/images/posts/reward-responsibility/building-blocks.jpg)

<small>각 블록이 하나의 책임을 갖는 구조. Photo by [Duc Anh Nguyen on Pexels](https://www.pexels.com/photo/colorful-building-blocks-on-a-table-8289844/).</small>

<div class="notice--info" markdown="1">
**AI 대화와 보상 파이프라인 개발기**

1. [AI 대화 후처리 개선기](/posts/ai-conversation-post-processing-pipeline/)
2. **리워드 생성·수령·충전 책임 분리** — 현재 글
3. [상태 조건 UPDATE로 중복 수령 방지](/posts/reward-conditional-update-idempotency/)
</div>

---

## 1. 사고를 한 장으로 요약하면

운영에서 기대한 변화는 한 번의 충전이었지만 실제로는 두 번 반영됐습니다.

~~~mermaid
sequenceDiagram
    autonumber
    participant E as 보상 생성 경로
    participant R as 보상 수령 경로
    participant C as 두리안 충전
    participant B as 사용자 잔액

    E->>C: 동일 보상으로 금액 충전
    C->>B: 잔액 반영
    R->>C: 동일 보상으로 다시 충전
    C->>B: 잔액 반영
    Note over B: 기대 1회 / 실제 2회
~~~

처음에는 “충전 메서드가 두 번 호출됐다”가 원인처럼 보였습니다. 더 정확한 원인은 다음과 같았습니다.

> 하나의 보상에 대해 충전을 시작할 수 있는 책임이 두 경로에 동시에 존재했다.

| 관찰한 현상 | 표면적 처방 | 구조적 처방 |
|---|---|---|
| 같은 보상으로 두 번 충전 | 한쪽 호출 삭제 | 충전 시작 권한을 한 흐름에만 둔다 |
| 생성과 지급의 의미 혼재 | 메서드 이름 변경 | 상태와 유스케이스를 분리한다 |
| 수령 API 재호출 | 컨트롤러 중복 방지 | DB 상태 전이를 성공 조건으로 사용한다 |
| 잔액만 남아 원인 추적 어려움 | 로그 추가 | 재화 원장에 출처를 기록한다 |

---

## 2. 처음 설계가 틀렸다기보다 요구사항이 달라졌다

초기 조건형 보상은 “조건 달성 즉시 지급”이었습니다.

~~~mermaid
flowchart LR
    C["업무 조건 계산"] --> P["보상 상품 조회"]
    P --> R["Reward 생성"]
    R --> D["두리안 즉시 충전"]
    D --> B["잔액 반영"]
~~~

당시 <code>CreateRewardService</code>는 다음 책임을 함께 가지고 있었습니다.

1. 보상 조건을 계산한다.
2. 보상 상품을 선택한다.
3. Reward 레코드를 저장한다.
4. 사용자 두리안 잔액을 증가시킨다.
5. 두리안 거래 내역을 기록한다.

사용자가 별도의 수령 행동을 하지 않는 요구사항에서는 합리적인 흐름이었습니다. 이후 UI에 “받기”와 “모두 받기”가 생기면서 리워드는 즉시 지급 결과가 아니라 수령 가능한 권리가 됐습니다.

~~~text
기존 의미  Reward = 지급된 재화의 기록
변경 의미  Reward = 사용자가 수령할 수 있는 권리
~~~

의미가 달라졌는데 기존 충전 경로를 그대로 남겨두면 생성과 수령 양쪽이 같은 부수효과를 가질 수밖에 없습니다.

---

## 3. 리워드와 두리안은 같은 도메인이 아니다

책임을 분리하기 위해 먼저 두 개념을 구분했습니다.

| 도메인 | 핵심 질문 | 대표 데이터 |
|---|---|---|
| Claim | 왜, 어떤 조건으로, 어떤 보상을 받을 권리가 생겼는가? | kind, status, itemId |
| Asset | 실제 잔액이 얼마나 변했고 그 출처는 무엇인가? | balance, ledger, source |

~~~mermaid
classDiagram
    class Claim {
        UUID id
        UUID subjectId
        ClaimKind kind
        ClaimStatus status
        UUID itemId
    }

    class RewardItem {
        UUID id
        int durianBase
        int durianBonus
        int duration
    }

    class AssetBalance {
        UUID subjectId
        int paid
        int free
        long version
    }

    class AssetLedger {
        UUID sourceId
        LedgerSourceType sourceType
        int amount
        int balanceBefore
        int balanceAfter
    }

    Claim --> RewardItem : 어떤 상품인가
    AssetLedger --> Claim : sourceId
    AssetBalance --> AssetLedger : 잔액 변화의 원장
~~~

Claim은 재화 그 자체가 아닙니다. “상품 X를 받을 수 있는 권리”를 나타냅니다. 실제 재화 변화는 잔액과 원장이 담당합니다.

<div class="notice--info" markdown="1">
**분리 후의 핵심 문장**

- 생성은 권리를 만든다.
- 수령은 권리의 상태를 전이한다.
- 충전은 전이된 권리를 잔액과 원장에 반영한다.
</div>

---

## 4. 목표 구조: 세 유스케이스, 하나의 방향

~~~mermaid
flowchart LR
    subgraph Create["1. 생성"]
        C1["조건 검증"] --> C2["Reward<br/>PENDING"]
    end

    subgraph Receive["2. 수령"]
        R1["소유권 검증"] --> R2["PENDING → RECEIVED"]
    end

    subgraph Charge["3. 충전"]
        D1["거래 원장 생성"] --> D2["사용자 잔액 증가"]
    end

    C2 --> R1
    R2 --> D1

    style Create fill:#e3f2fd,stroke:#1565c0
    style Receive fill:#fff8e1,stroke:#f9a825
    style Charge fill:#e8f5e9,stroke:#2e7d32
~~~

의존 방향도 한쪽으로만 흐르게 했습니다.

~~~text
CreateRewardUseCase
        ↓ PENDING Reward
ReceiveRewardUseCase
        ↓ 성공한 상태 전이
ChargeDurianUseCase
        ↓ 거래 원장 + 잔액
~~~

충전 서비스는 리워드가 수령 가능한지 판단하지 않습니다. 리워드 수령 서비스는 잔액을 직접 계산하지 않습니다. 생성 서비스는 사용자가 언제 수령했는지 결정하지 않습니다.

---

## 5. 생성 책임: 조건을 판정하고 PENDING을 만든다

조건형 보상 생성은 다음 흐름으로 처리합니다. 구체적인 트리거와 구간 값은 설정된 정책으로 치환했습니다.

~~~mermaid
flowchart TD
    A["조건형 보상 생성 시도"] --> B{동일 업무 키로 이미 생성됐는가?}
    B -->|예| X["스킵"]
    B -->|아니오| C["업무 조건 계산"]
    C --> D{지급 조건을 만족하는가?}
    D -->|아니오| X
    D -->|예| Q{제외 조건에 해당하는가?}
    Q -->|예| S["미지급 사유 기록"]
    Q -->|아니오| R1["정책에 맞는 Reward 생성"]
    S --> M["다음 정책 구간 확인"]
    R1 --> M
    M --> R2["추가 조건 충족 시 Reward 생성"]
~~~

실제 생성 메서드는 Reward를 저장한 뒤 결과만 반환합니다. 이 경로에는 두리안 충전 호출이 없습니다.

~~~kotlin
private fun createClaimIfEligible(
    command: CreateRewardCommand,
    policyKey: RewardPolicyKey,
): CommandRewardResult? {
    val policy = rewardPolicyPort.findByKey(policyKey) ?: return null

    val product =
        queryProductDurianPort.findById(
            policy.productId
        ) ?: return null

    val reward = Reward.create(
        userId = command.userId,
        type = RewardType.CLAIMABLE,
        durianProductId = product.id
    )

    val savedReward = commandRewardPort.saveReward(reward)

    return CommandRewardResult(
        id = savedReward.id,
        userId = savedReward.userId,
        type = savedReward.type,
        status = savedReward.status,
        durianProductId = savedReward.durianProductId,
        quantity = product.durianTotal,
        description = product.description,
        createdAt = savedReward.createdAt,
        updatedAt = savedReward.updatedAt
    )
}
~~~

<code>Reward.create</code>의 기본 상태는 <code>PENDING</code>입니다.

~~~mermaid
stateDiagram-v2
    [*] --> PENDING: 수령형 조건 달성
    PENDING --> RECEIVED: 사용자가 수령
    RECEIVED --> [*]
~~~

다른 수령형 보상도 같은 규칙을 따릅니다. 각 정책 조건과 기존 동일 업무 키 여부를 검사한 뒤 PENDING Reward만 생성합니다.

---

## 6. 수령 책임: 상태 전이에 성공한 요청만 충전을 시작한다

단건 수령 서비스는 다음 순서로 동작합니다.

~~~mermaid
sequenceDiagram
    autonumber
    actor U as 사용자
    participant R as ReceiveRewardService
    participant DB as Reward Repository
    participant C as ChargeDurianService
    participant L as Durian Ledger

    U->>R: rewardId 수령 요청
    R->>DB: Reward 조회
    R->>R: 소유권·현재 상태 검사
    R->>DB: PENDING 조건 UPDATE
    DB-->>R: affectedRows
    alt affectedRows = 1
        R->>C: rewardId로 충전 요청
        C->>L: 거래 원장 + 잔액 반영
        R-->>U: 수령 결과
    else affectedRows = 0
        R-->>U: 이미 수령된 보상
    end
~~~

~~~kotlin
@Transactional
override fun receiveReward(
    command: ReceiveRewardCommand
): ReceiveRewardsResult {
    val reward = queryRewardPort.findById(command.rewardId)
        ?: throw NotFoundRewardException()

    if (!reward.isOwnedBy(command.userId)) {
        throw AccessDeniedException()
    }

    if (reward.isAlreadyReceived()) {
        throw AlreadyReceivedRewardException()
    }

    val updatedCount =
        commandRewardPort.updateStatusToReceived(reward.id)

    if (updatedCount == 0) {
        throw AlreadyReceivedRewardException()
    }

    val updatedReward =
        queryRewardPort.findById(reward.id)
            ?: throw NotFoundRewardException()

    val balance =
        chargeDurianUseCase.chargeDurianByReward(
            ChargeDurianByRewardCommand(
                userId = command.userId,
                rewardId = updatedReward.id,
                productId = updatedReward.durianProductId,
                title = resolveTitleByRewardType(updatedReward.type)
            )
        )

    // updatedReward와 balance로 ReceiveRewardsResult를 구성한다.
}
~~~

위 코드는 응답 DTO 매핑을 제외한 실제 핵심 흐름입니다. 사전 상태 확인은 빠른 실패를 위한 사용자 친화적 검증입니다. 실제 중복 방지의 최종 판단은 DB의 조건부 UPDATE 결과가 담당합니다. 이 부분은 세 번째 글에서 동시성 타임라인과 함께 자세히 다룹니다.

---

## 7. 충전 책임: 원장을 먼저 만들고 잔액을 갱신한다

<code>ChargeDurianService</code>는 보상 외에도 주문, 관리자 지급, 쿠폰 등 여러 출처의 충전을 처리합니다.

~~~mermaid
flowchart LR
    O["유료 주문"] --> C["ChargeDurianService"]
    R["리워드 수령"] --> C
    A["관리자 지급"] --> C
    P["쿠폰"] --> C
    C --> T["AssetLedger 저장"]
    C --> U["UserDurian 잔액 저장"]
~~~

공통 충전 로직은 상품을 조회하고, 현재 잔액을 기준으로 거래 원장을 만든 뒤, 무료 또는 유료 잔액을 증가시킵니다.

~~~kotlin
private fun processingDurian(
    productId: UUID,
    userId: UUID,
    durianType: DurianType,
    sourceType: LedgerSourceType,
    sourceId: UUID,
    meta: String?,
): GetUserDurianBalanceResult {
    val productDurian = queryProductDurianPort.findById(productId)
        ?: throw NotFoundProductException()

    val userDurian = queryUserDurianPort.findByUserId(userId)
        ?: UserDurian.create(userId)

    val currentBalancePaid = userDurian.balancePaid
    val currentBalanceFree = userDurian.balanceFree
    val currentTotalBalance = userDurian.balanceTotal

    val totalAmount =
        productDurian.durianBase + productDurian.durianBonus
    val expiresAt = TimezoneConverter.calculateExpiresAtInUtc(
        productDurian.duration.toLong()
    )

    val chargeTransaction = AssetLedger.createCredit(
        userId = userId,
        durianType = durianType,
        sourceType = sourceType,
        sourceId = sourceId,
        amount = totalAmount,
        balanceBefore = currentTotalBalance,
        expiresAt = expiresAt,
        meta = meta
    )

    commandAssetLedgerPort.save(chargeTransaction)

    val updatedUserDurian = if (durianType.isFree()) {
        userDurian.copy(
            balanceFree = currentBalanceFree + totalAmount
        )
    } else {
        userDurian.copy(
            balancePaid = currentBalancePaid + totalAmount
        )
    }
    commandUserDurianPort.save(updatedUserDurian)

    return GetUserDurianBalanceResult(
        balancePaid = updatedUserDurian.balancePaid,
        balanceFree = updatedUserDurian.balanceFree,
        balanceTotal = updatedUserDurian.balanceTotal
    )
}
~~~

### sourceType과 sourceId가 남기는 것

| 필드 | 보상 충전 예시 | 의미 |
|---|---|---|
| sourceType | REWARD | 어떤 도메인에서 시작됐는가 |
| sourceId | reward.id | 어떤 보상이 원인이었는가 |
| entryType | CREDIT | 증가·차감 중 무엇인가 |
| amount | 정책으로 결정 | 얼마가 변했는가 |
| balanceBefore / After | 계산된 값 | 전후 잔액 |

이 정보 덕분에 잔액 숫자 하나만 보는 대신 “어떤 보상 때문에 언제 얼마나 증가했는지”를 추적할 수 있습니다.

<div class="notice--warning" markdown="1">
**현재 제약의 정확한 범위**

거래 엔티티에는 출처와 거래 종류를 묶은 유일 제약이 선언되어 있지 않습니다. 따라서 원장 자체가 보상 충전의 중복을 최종 차단하는 구조는 아닙니다. 현재의 1차 중복 차단선은 Reward의 PENDING → RECEIVED 조건부 UPDATE입니다.
</div>

---

## 8. 책임 분리와 트랜잭션 분리는 같은 말이 아니다

서비스 클래스는 세 개로 나눴지만 단건 수령은 하나의 Spring 트랜잭션 안에서 처리됩니다.

~~~mermaid
flowchart LR
    subgraph TX["하나의 수령 트랜잭션"]
        A["Reward 상태 전이"] --> B["두리안 원장 저장"]
        B --> C["사용자 잔액 저장"]
    end

    C --> D{모두 성공?}
    D -->|예| E["COMMIT"]
    D -->|아니오| F["ROLLBACK"]
~~~

<code>ReceiveRewardService.receiveReward</code>와 <code>ChargeDurianService.chargeDurianByReward</code>는 서로 다른 Spring 빈입니다. 두 메서드 모두 기본 전파 속성의 <code>@Transactional</code>을 사용하므로 충전 서비스는 바깥 수령 트랜잭션에 참여합니다.

| 실패 지점 | Reward 상태 | 거래 원장 | 사용자 잔액 |
|---|---|---|---|
| 조건부 UPDATE 실패 | 변경 없음 | 생성 안 됨 | 변경 없음 |
| 상품 조회 실패 | 롤백 | 생성 안 됨 | 변경 없음 |
| 원장 저장 실패 | 롤백 | 롤백 | 변경 없음 |
| 잔액 저장 실패 | 롤백 | 롤백 | 롤백 |
| 모든 단계 성공 | RECEIVED | 커밋 | 커밋 |

책임은 분리하되 원자적으로 함께 성공해야 하는 상태 변화는 같은 트랜잭션에 남겼습니다.

> 클래스 경계는 “누가 무엇을 아는가”를 나누고, 트랜잭션 경계는 “무엇이 함께 성공해야 하는가”를 묶습니다.

---

## 9. 일괄 수령은 충전 서비스에도 별도 유스케이스가 필요했다

여러 보상을 하나씩 충전하면 사용자 잔액을 반복 조회·저장하고 거래도 건별 호출하게 됩니다. 그래서 일괄 수령은 성공한 보상 목록을 하나의 벌크 커맨드로 넘깁니다.

~~~mermaid
flowchart TD
    P["PENDING 보상 N개 조회"] --> U["조건부 일괄 상태 UPDATE"]
    U --> S["성공한 보상 구성"]
    S --> T["N개 거래 원장 객체 생성"]
    T --> TS["거래 원장 saveAll"]
    TS --> B["최종 잔액 한 번 저장"]
~~~

~~~kotlin
@Transactional
override fun chargeDurianBulk(
    command: ChargeDurianBulkCommand
): GetUserDurianBalanceResult {
    val products = queryProductDurianPort
        .findAllByIdIn(command.items.map { it.productId }.distinct())
        .associateBy { it.id }

    val userDurian =
        queryUserDurianPort.findByUserId(command.userId)
            ?: UserDurian.create(command.userId)

    var paid = userDurian.balancePaid
    var free = userDurian.balanceFree
    var total = userDurian.balanceTotal

    val transactions = mutableListOf<AssetLedger>()

    for (item in command.items) {
        val product = products[item.productId]
            ?: throw NotFoundProductException()
        val amount = product.durianBase + product.durianBonus
        val expiresAt = TimezoneConverter.calculateExpiresAtInUtc(
            product.duration.toLong()
        )

        transactions += AssetLedger.createCredit(
            userId = command.userId,
            durianType = item.durianType,
            sourceType = item.sourceType,
            sourceId = item.sourceId,
            amount = amount,
            balanceBefore = total,
            expiresAt = expiresAt,
            meta = item.title?.toMetaJson()
        )

        if (item.durianType.isFree()) free += amount
        else paid += amount
        total += amount
    }

    commandAssetLedgerPort.saveAll(transactions)
    val updatedUserDurian = commandUserDurianPort.save(
        userDurian.copy(balancePaid = paid, balanceFree = free)
    )

    return GetUserDurianBalanceResult(
        balancePaid = updatedUserDurian.balancePaid,
        balanceFree = updatedUserDurian.balanceFree,
        balanceTotal = updatedUserDurian.balanceTotal
    )
}
~~~

| 항목 | N번 단건 충전 | 벌크 충전 |
|---|---:|---:|
| 상품 조회 | 최대 N회 | 고유 상품 ID 한 번 |
| 사용자 잔액 조회 | N회 | 한 번 |
| 잔액 저장 | N회 | 한 번 |
| 원장 데이터 | N건 | N건 유지 |
| 트랜잭션 경계 | 반복 가능 | 한 번 |

원장은 N건을 유지합니다. 일괄 처리라고 해서 원인 추적 단위를 잃지 않으면서, 잔액 계산과 저장만 묶었습니다.

---

## 10. 모든 보상이 같은 정책을 따르는 것은 아니다

현재 코드에는 사용자가 직접 수령하는 보상과 조건 달성 즉시 지급되는 보상이 함께 있습니다. 실제 보상 트리거 이름은 공개용 예시에서 A·B·C 유형으로 일반화했습니다.

~~~mermaid
flowchart LR
    subgraph Claimable["사용자 수령형"]
        C["조건형 A·B"] --> P["PENDING"]
        P --> R["수령"]
        R --> D["충전"]
    end

    subgraph Immediate["즉시 지급형"]
        O["조건형 C"] --> I["RECEIVED 생성"]
        I --> IC["즉시 충전"]
    end
~~~

~~~kotlin
val reward = Reward.create(
    userId = command.userId,
    type = RewardType.IMMEDIATE,
    durianProductId = product.id,
    status = RewardStatus.RECEIVED
)

val savedReward = commandRewardPort.saveReward(reward)

chargeDurianUseCase.chargeDurianByReward(
    ChargeDurianByRewardCommand(
        userId = command.userId,
        rewardId = savedReward.id,
        productId = product.id,
        title = LedgerTitle.REWARD_IMMEDIATE
    )
)
~~~

이 경로까지 포함해 “모든 Reward 생성은 충전과 완전히 분리됐다”고 말하면 현재 구현과 다릅니다.

| 보상 타입 | 생성 상태 | 사용자 수령 필요 | 충전 시작 위치 |
|---|---|---:|---|
| CLAIMABLE_A | PENDING | 예 | ReceiveRewardService |
| CLAIMABLE_B | PENDING | 예 | ReceiveRewardService |
| IMMEDIATE_C | RECEIVED | 아니오 | CreateRewardService |

책임 분리는 코드 모양을 통일하는 일이 아니라 제품 정책을 정확히 드러내는 일입니다. 즉시 지급형과 수령형이 모두 필요하다면 타입이나 지급 정책을 명시적으로 모델링하는 편이 다음 개선 방향입니다.

---

## 11. 포트로 경계를 고정했다

핵심 서비스는 영속성 구현과 직접 결합하지 않습니다.

~~~mermaid
flowchart LR
    subgraph Core["Core"]
        CR["CreateRewardService"]
        RR["ReceiveRewardService"]
        CD["ChargeDurianService"]
        CP["CommandRewardPort"]
        QP["QueryRewardPort"]
        DP["CommandUserDurianPort"]
        TP["CommandAssetLedgerPort"]
    end

    subgraph Infra["PostgreSQL Adapter"]
        RA["Reward Adapter"]
        DA["UserDurian Adapter"]
        TA["Transaction Adapter"]
    end

    CR --> CP
    RR --> QP
    RR --> CP
    RR --> CD
    CD --> DP
    CD --> TP
    CP -.-> RA
    QP -.-> RA
    DP -.-> DA
    TP -.-> TA
~~~

| 서비스 | 알아야 하는 것 | 몰라도 되는 것 |
|---|---|---|
| CreateRewardService | 보상 조건, 상품, 초기 상태 | 잔액 계산 방식 |
| ReceiveRewardService | 소유권, 상태 전이, 충전 유스케이스 | JPA 저장 구현 |
| ChargeDurianService | 상품 수량, 원장, 잔액 | 보상 생성 조건 |

이 경계 덕분에 조건부 UPDATE는 Reward 어댑터에, 잔액 계산은 충전 서비스에, 오케스트레이션은 수령 서비스에 둘 수 있었습니다.

---

## 12. 분리 후에도 남은 동시성 과제

책임을 분리하면 중복 충전 지점은 줄어들지만 동시성 문제가 자동으로 사라지는 것은 아닙니다.

### 12.1 보상 생성의 선조회 경쟁

오늘 보상 존재 여부를 조회한 뒤 새 Reward를 저장하는 구조는 동시에 두 요청이 들어오면 둘 다 “없음”을 볼 수 있습니다. Reward 테이블에 업무 키 유일 제약이 없다면 중복 생성 가능성이 남습니다.

### 12.2 거래 원장의 유일 제약 부재

같은 rewardId를 원인으로 하는 CREDIT 원장이 두 번 생성되는 것을 DB 제약으로 막지 않습니다. 수령 상태 전이가 뚫리거나 다른 즉시 지급 경로가 추가되면 원장 계층만으로는 방어하지 못합니다.

### 12.3 사용자 잔액의 동시 갱신

<code>UserDurianEntity</code>에는 <code>@Version</code>이 있어 잃어버린 갱신을 감지합니다. 다만 충돌 시 어떤 요청을 재시도할지에 대한 정책은 별도로 필요합니다.

~~~mermaid
flowchart TD
    G["중복 방지 레이어"] --> A["생성: 업무 키 UNIQUE"]
    G --> B["수령: 상태 조건 UPDATE"]
    G --> C["원장: source 유일 키"]
    G --> D["잔액: version 또는 잠금"]

    style B fill:#c8e6c9,stroke:#2e7d32,color:#111
    style A fill:#fff9c4,stroke:#f9a825,color:#111
    style C fill:#fff9c4,stroke:#f9a825,color:#111
    style D fill:#e3f2fd,stroke:#1565c0,color:#111
~~~

현재 구현은 B를 핵심 게이트로 사용하고 D에서 낙관적 잠금을 적용합니다. A와 C를 DB 제약으로 보강하면 한 계층의 실수가 실제 재화 중복으로 이어질 가능성을 더 낮출 수 있습니다.

---

## 13. 배운 점

### 13.1 중복 호출보다 중복 책임을 먼저 찾아야 한다

한 줄을 삭제하면 당장의 중복 충전은 멈춥니다. 하지만 다음 보상 타입이 같은 실수를 반복할 수 있습니다. 누가 충전을 시작할 수 있는지 한 곳으로 제한한 것이 더 근본적인 수정이었습니다.

### 13.2 상태는 UI 표시값이 아니라 부수효과의 게이트다

PENDING과 RECEIVED는 목록 화면을 위한 라벨이 아닙니다. 충전을 시작할 수 있는 요청을 결정하는 업무 규칙입니다.

### 13.3 책임을 나눠도 원자성은 유지할 수 있다

생성·수령·충전을 서로 다른 서비스로 나눴지만 수령 상태, 원장, 잔액은 같은 트랜잭션에서 커밋됩니다. 모듈성과 원자성은 서로 반대되는 목표가 아닙니다.

### 13.4 원장은 잔액만큼 중요하다

재화 시스템에서 “현재 얼마인가”만으로는 운영 사고를 분석할 수 없습니다. sourceType과 sourceId를 포함한 거래 원장이 있어야 변화의 원인을 역추적할 수 있습니다.

### 13.5 예외 정책은 숨기지 말아야 한다

일부 보상은 여전히 즉시 지급입니다. 모든 타입을 억지로 같은 흐름에 넣기보다 어떤 정책이 왜 예외인지 드러내는 편이 유지보수에 유리합니다.

---

## 마무리

리워드 생성과 두리안 충전을 한 메서드에서 처리하던 구조는 초기 요구사항에는 충분했습니다. 그러나 수령 UI와 여러 보상 타입이 추가되면서 “생성 즉시 지급”이라는 암묵적 의미가 더 이상 맞지 않았습니다.

Reward를 받을 권리, Receive를 상태 전이의 오케스트레이션, Charge를 원장과 잔액 변경으로 다시 정의하면서 충전 시작점은 수령 흐름으로 모였습니다. 클래스는 나뉘었지만 함께 성공해야 하는 상태 변화는 하나의 트랜잭션으로 묶었습니다.

도메인 책임 분리의 효과는 파일 수가 늘어나는 데 있지 않습니다. 같은 부수효과를 시작할 수 있는 이유와 장소가 하나로 줄어드는 데 있습니다.

---

## 참고 자료
- [Pexels 이미지 원문 — Duc Anh Nguyen](https://www.pexels.com/photo/colorful-building-blocks-on-a-table-8289844/)

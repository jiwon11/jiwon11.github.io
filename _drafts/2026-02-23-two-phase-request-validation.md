---
title: "프로필 열람 요청의 이행을 커밋 이후로 미루고 두 시점에서 검증한 이유"
date: 2026-02-23
category: 개발기
tags: [spring-boot, kotlin, event-driven, transaction, validation, consistency]
readTime: 15min
mermaid: true
image:
  path: "https://images.unsplash.com/photo-1544383835-bda2bc66a55d?auto=format&fit=crop&w=1200&q=80"
  thumbnail: "https://images.unsplash.com/photo-1544383835-bda2bc66a55d?auto=format&fit=crop&w=600&q=80"
  alt: "칸마다 이름표가 붙은 서랍장"
---

요청을 받아 바로 처리하지 않고 나중에 이행하는 기능이 있습니다. 사용자가 먼저 요청을 남겨두면, 조건이 갖춰졌을 때 서버가 이벤트를 받아 이행하는 구조입니다.

이런 기능에서는 요청을 받는 시점과 이행하는 시점 사이에 시간이 흐릅니다. 그 사이에 요청의 근거가 되던 관계가 사라질 수 있고, 이행을 촉발한 작업 자체가 롤백될 수도 있습니다. 이번 글에서는 이행을 트랜잭션 커밋 이후로 미루고, 생성 시점과 이행 시점 두 곳에서 각각 검증하도록 정리한 과정을 다뤄보겠습니다.

> **예제 코드 안내**
>
> 아래의 클래스·메서드·상태·에러 이름과 API 경로는 모두 가상 예시입니다. 실제 구현의 상태 전이, 관계 종류, 응답 규격, 알림 정책과 대응되지 않습니다.
{: .prompt-info }

---

## 1. 존재만 확인하고 상태는 보지 않던 코드

사용자는 서로 관계를 맺은 상대에게만 프로필 열람을 요청할 수 있습니다. 관계는 세 가지 경로로 생깁니다. 그룹에 함께 속하면서 맺어지는 관계, 메시지를 주고받으며 이어지는 관계, 서버가 상대를 제안하고 사용자가 응답하면서 맺어지는 관계입니다.

기존 코드는 어떤 경로로 관계가 생겼는지 판별하기 위해 각 저장소를 순서대로 조회했습니다.

~~~kotlin
private fun resolveSource(requesterId: UUID, targetUserId: UUID): RequestSource {
    val groups = queryGroupPort.findAllByUserId(requesterId)
    val inGroup = groups.any { group ->
        queryMemberPort.findByGroupId(group.id).any { it.contains(targetUserId) }
    }
    if (inGroup) return RequestSource.GROUP

    val proposal = queryProposalPort.find(requesterId, targetUserId)
        ?: queryProposalPort.find(targetUserId, requesterId)
    if (proposal != null) return RequestSource.PROPOSAL

    return RequestSource.PROPOSAL
}
~~~

이 코드는 관계의 **존재 여부**만 확인하고 **상태**는 확인하지 않습니다. 그래서 다음 세 가지 경우에 무효한 요청이 통과합니다.

| 상황 | 결과 |
|---|---|
| 그룹이 취소된 뒤에도 조회에는 남아 있음 | 취소된 상대에게 요청 발송 |
| 제안을 거절한 뒤에도 제안 레코드는 남아 있음 | 거절한 상대에게 요청 발송 |
| 아무 관계가 없어도 마지막 줄이 기본값을 반환 | 무관한 사용자에게 요청 발송 |

특히 마지막 줄이 문제였습니다. 판별에 실패한 경우와 실제로 제안 관계인 경우가 **같은 값으로 반환**되어, 관계가 없는 사용자에게도 요청이 만들어졌습니다. 조회 결과가 비어 있을 때 기본값을 돌려주는 코드는 호출한 쪽에서 실패를 알아챌 방법이 없습니다.

---

## 2. 이행이 커밋보다 먼저 일어나던 문제

요청은 즉시 이행되지 않습니다. 대상자의 프로필이 만들어지고 공개로 전환되면 그때 이벤트가 발행되고, 대기 중이던 요청이 이행되면서 알림이 나갑니다.

문제는 이벤트를 발행하는 위치였습니다. 프로필 생성 로직 안에서 이벤트를 발행하면, 그 이벤트는 **생성 트랜잭션이 아직 커밋되지 않은 상태**에서 처리됩니다.

~~~mermaid
sequenceDiagram
    autonumber
    participant S as 생성 트랜잭션
    participant L as 리스너
    participant R as 요청자

    S->>S: 프로필 저장
    S->>L: 이벤트 발행
    L->>R: 요청 이행 및 알림 발송
    S--xS: 이후 단계에서 예외 발생, 롤백
    Note over R: 존재하지 않는 프로필에 대한 알림만 남음
~~~

알림은 이미 나갔는데 프로필은 롤백되어 사라집니다. 사용자는 알림을 받고 들어왔지만 볼 것이 없습니다. 알림은 외부로 나가는 작업이라 트랜잭션과 함께 되돌릴 수 없기 때문에, **애초에 커밋이 확정된 뒤에만 실행되어야 하는 작업**이었습니다.

---

## 3. 검증 지점이 두 곳이어야 하는 이유

이행 시점을 커밋 이후로 옮기면 롤백 문제는 해결됩니다. 그런데 검증 문제는 그대로 남습니다.

요청과 이행 사이에는 시간 제한이 없습니다. 그동안 그룹이 취소되거나 제안이 거절될 수 있고, 요청자가 탈퇴할 수도 있습니다. 생성 시점에만 검증하면 **요청할 당시에는 유효했지만 이행할 때는 무효한 요청**이 그대로 이행되어, 이미 관계가 끊긴 상대에게 프로필이 노출됩니다.

반대로 이행 시점에만 검증하는 방법도 생각해볼 수 있습니다. 이 경우 무효한 요청이 만료 처리되기는 하지만, 애초에 근거 없는 요청이 저장소에 계속 쌓입니다. 사용자도 요청이 실패했다는 사실을 이행 시점까지 알 수 없습니다.

두 시점은 막는 대상이 다릅니다.

- **생성 시점 검증** — 근거 없는 요청이 저장되는 것을 막고, 사용자에게 즉시 실패를 알림
- **이행 시점 검증** — 저장된 이후에 무효해진 요청이 이행되는 것을 막음

한쪽만으로는 다른 쪽을 대신할 수 없어 두 시점 모두에서 검증하기로 했습니다.

---

## 4. 생성 시점 — 판별과 검증을 함께 수행하기

기존 메서드는 이름 그대로 경로를 판별하기만 했습니다. 여기에 상태 조건을 추가하고, 판별에 실패했을 때 기본값 대신 예외를 던지도록 바꿨습니다.

~~~kotlin
private fun validateAndResolveSource(requesterId: UUID, targetUserId: UUID): RequestSource {
    var groupExists = false
    val groups = queryGroupPort.findAllByUserId(requesterId)
    for (group in groups) {
        if (queryMemberPort.findByGroupId(group.id).any { it.contains(targetUserId) }) {
            groupExists = true
            val detail = queryGroupDetailPort.findByGroupId(group.id)
            if (detail != null && detail.status != GroupStatus.CANCELED) {
                return RequestSource.GROUP
            }
        }
    }

    val thread = queryThreadPort.findBetween(requesterId, targetUserId)
    if (thread != null && thread.status == ThreadStatus.OPEN) {
        return RequestSource.MESSAGE
    }

    var proposalExists = false
    val proposal = queryProposalPort.find(requesterId, targetUserId)
        ?: queryProposalPort.find(targetUserId, requesterId)
    if (proposal != null) {
        proposalExists = true
        val status = queryProposalDetailPort.find(proposal.id, proposal.userId)?.status
        if (status == null || status == ProposalStatus.ACCEPTED) {
            return RequestSource.PROPOSAL
        }
    }

    if (groupExists) throw InvalidGroupSourceException()
    if (thread != null) throw InvalidMessageSourceException()
    if (proposalExists) throw InvalidProposalSourceException()
    throw NoValidSourceException()
}
~~~

세 가지를 바꿨습니다.

**첫째, 상태 조건을 추가했습니다.** 그룹은 취소되지 않은 경우만, 메시지는 대화가 열려 있는 경우만, 제안은 아직 응답하지 않았거나 수락한 경우만 유효한 근거로 인정합니다.

**둘째, 관계의 존재와 유효를 나눠서 기록했습니다.** `groupExists`와 `proposalExists`를 따로 둔 이유는 실패 사유를 구분하기 위해서입니다. 관계가 아예 없는 경우와 관계는 있지만 취소된 경우는 사용자가 취해야 할 행동이 다릅니다. 전자는 잘못된 요청이고 후자는 이미 끝난 관계입니다. 한 가지 예외로 뭉뚱그리면 클라이언트가 안내 문구를 구분할 수 없습니다.

**셋째, 기본값 반환을 제거했습니다.** 판별에 실패하면 예외로 끝납니다. 조회에 실패했을 때 그럴듯한 값을 돌려주는 대신 실패를 실패로 알리는 편이, 호출하는 쪽에서 처리하기 쉽습니다.

### 검증 순서를 관계의 유효 기간 기준으로 정하기

한 사용자가 여러 경로로 동시에 관계를 맺고 있을 수 있습니다. 같은 그룹에 속하면서 메시지도 주고받는 경우입니다. 이때 어떤 경로를 근거로 기록하느냐에 따라 나중에 이행될지가 달라집니다.

순서는 **관계가 유지되는 기간이 긴 것부터** 확인하도록 정했습니다.

| 순서 | 관계 | 유효 기간 |
|---|---|---|
| 1 | 그룹 | 그룹이 취소되기 전까지 유지 |
| 2 | 메시지 | 대화가 이어지는 동안 유지 |
| 3 | 제안 | 응답하거나 만료되면 종료 |

짧게 끝나는 관계를 근거로 잡으면, 요청은 성공했는데 이행 시점에는 이미 만료되어 있을 확률이 높아집니다. 가장 오래 유지되는 관계를 먼저 근거로 잡아야 요청이 살아남을 가능성이 커집니다. 반대 순서로 두면 그룹 관계가 멀쩡한데도 제안이 먼저 걸려서 요청이 만료되는 일이 생깁니다.

---

## 5. 이행 시점 — 커밋 이후로 미루고 다시 검증하기

### 이벤트를 커밋 이후에 처리하기

먼저 롤백 문제를 해결하기 위해 리스너를 `AFTER_COMMIT` 단계에서 동작하도록 바꿨습니다.

~~~kotlin
@Component
class ProfileRevealedListener(
    private val validator: RequestSourceValidator,
    transactionManager: PlatformTransactionManager,
) {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handle(event: ProfileRevealedEvent) {
        // 커밋이 확정된 뒤에만 실행된다
    }
}
~~~

이제 프로필 생성이 롤백되면 이벤트 자체가 처리되지 않습니다. 알림이 나갔는데 프로필이 없는 상황이 구조적으로 생기지 않습니다.

다만 `AFTER_COMMIT` 시점에는 원래 트랜잭션이 이미 종료돼 있습니다. 이 시점에 기본 전파 속성(`REQUIRED`)으로 저장하면 이미 끝난 트랜잭션에 참여하게 되어 변경 내용이 커밋되지 않습니다. 그래서 리스너 안의 저장 작업은 **새 트랜잭션 경계**에서 실행해야 합니다.

~~~kotlin
private val requiresNew = TransactionTemplate(transactionManager).apply {
    propagationBehavior = TransactionDefinition.PROPAGATION_REQUIRES_NEW
}

private fun saveInNewTransaction(requests: List<ViewRequest>) {
    requiresNew.executeWithoutResult {
        commandRequestPort.saveAll(requests)
    }
}
~~~

공유 `TransactionTemplate` 빈의 전파 속성을 바꾸면 같은 빈을 쓰는 다른 코드의 동작까지 바뀌므로, `REQUIRES_NEW` 전용 템플릿을 따로 만들었습니다.

`REQUIRES_NEW`로 분리하면 요청 상태 갱신이 자체 트랜잭션에서 커밋됩니다. 프로필 생성은 이미 커밋된 뒤이므로, 이 저장이 실패해도 프로필 생성이 되돌려지지는 않습니다.

### 이행 직전에 다시 검증하기

이행 시점에도 관계 판정이 필요합니다. 다만 필요한 답이 다릅니다. 생성 시점에는 "어떤 경로인가"를 알아야 하고, 이행 시점에는 이미 저장된 경로가 "아직 유효한가"만 알면 됩니다.

그래서 유효성 판정만 담당하는 컴포넌트를 분리했습니다. 여기에는 관계뿐 아니라 **요청자가 아직 활성 상태인지**도 함께 확인합니다. 요청을 남긴 뒤 탈퇴한 사용자에게 알림을 보낼 이유가 없기 때문입니다.

~~~kotlin
@Component
class RequestSourceValidator(
    private val queryUserPort: QueryUserPort,
    private val queryGroupPort: QueryGroupPort,
    private val queryThreadPort: QueryThreadPort,
    private val queryProposalPort: QueryProposalPort,
) {
    fun isStillValid(request: ViewRequest): Boolean {
        if (!queryUserPort.isActive(request.requesterId)) return false

        return when (request.source) {
            RequestSource.GROUP -> isGroupValid(request.requesterId, request.targetUserId)
            RequestSource.MESSAGE -> isThreadOpen(request.requesterId, request.targetUserId)
            RequestSource.PROPOSAL -> isProposalValid(request.requesterId, request.targetUserId)
        }
    }
}
~~~

리스너는 대기 중인 요청을 유효한 것과 무효한 것으로 나눈 뒤, 무효한 요청은 이행하지 않고 만료 처리합니다.

~~~kotlin
val (valid, invalid) = pendingRequests.partition { validator.isStillValid(it) }

if (invalid.isNotEmpty()) {
    saveInNewTransaction(invalid.map { it.expire() })
    invalid.forEach { log.info { "근거 관계 만료로 요청 만료 처리 - source=${it.source}" } }
}

if (valid.isEmpty()) return

saveInNewTransaction(valid.map { it.fulfill() })
// 이후 알림 발송
~~~

무효한 요청을 그냥 넘기지 않고 **만료 상태로 남기는 것**이 중요했습니다. 대기 상태로 두면 다음 이벤트에서 다시 검사 대상이 되고, 지워버리면 왜 이행되지 않았는지 확인할 수 없습니다. 만료 상태로 기록하면 이후에 문의가 들어와도 요청이 접수됐고 관계가 끊겨 이행되지 않았다는 사실을 그대로 설명할 수 있습니다.

로그도 같은 이유로 남겼습니다. 알림이 오지 않았다는 문의가 들어왔을 때, 확인할 지점이 없으면 원인을 재현하기 어렵습니다.

---

## 6. 두 시점을 시나리오로 나눠 확인하기

검증 지점이 두 개라 조합해야 할 경우의 수가 늘어납니다. 관계 종류 세 가지, 상태 유효와 무효, 검증 시점 두 가지가 곱해집니다. 그래서 시나리오를 표로 먼저 정리한 뒤 하나씩 호출해 확인했습니다.

**생성 시점** — 관계 상태에 따라 요청이 저장되는지 확인합니다.

| 관계 | 상태 | 기대 결과 |
|---|---|---|
| 그룹 | 활성 | 요청 저장 |
| 그룹 | 취소 | 취소 사유로 실패 |
| 메시지 | 열림 | 요청 저장 |
| 메시지 | 종료 | 종료 사유로 실패 |
| 제안 | 미응답 | 요청 저장 |
| 제안 | 수락 | 요청 저장 |
| 제안 | 거절 | 거절 사유로 실패 |
| 없음 | — | 근거 없음으로 실패 |
| 제안 | 미응답 (중복 요청) | 중복 사유로 실패 |

마지막 행은 같은 상대에게 이미 대기 중인 요청이 있는데 다시 요청한 경우입니다.

**이행 시점** — 요청을 저장한 뒤 관계를 무효화하거나 생성을 실패시키고, 결과를 확인합니다.

| 요청 후 변화 | 기대 결과 |
|---|---|
| 변화 없음 | 이행 및 알림 발송 |
| 그룹이 취소로 변경 | 만료 처리, 알림 없음 |
| 메시지 대화가 종료됨 | 만료 처리, 알림 없음 |
| 제안이 거절로 변경 | 만료 처리, 알림 없음 |
| 요청자가 탈퇴 | 만료 처리, 알림 없음 |
| 프로필 생성이 롤백됨 | 이벤트 미처리, 알림 없음 |

이행 시점 확인은 순서가 중요했습니다. 프로필을 비공개로 둔 상태에서 요청을 저장하고, 그다음 관계를 무효로 바꾸고, 마지막에 공개로 전환해야 실제와 같은 시간 순서가 만들어집니다. 순서를 바꾸면 생성 시점 검증에 먼저 걸려서 이행 시점 동작을 확인할 수 없습니다.

롤백 시나리오는 프로필 저장 이후 단계에서 강제로 예외를 발생시켜, 알림이 나가지 않고 요청이 대기 상태로 남는지 확인했습니다.

개선 전에는 통과하던 무효 요청이 개선 후에는 모두 실패나 만료로 처리되었습니다.

| 시나리오 | 개선 전 | 개선 후 |
|---|---|---|
| 취소된 그룹으로 요청 | 요청 성공 | 실패 |
| 종료된 대화로 요청 | 요청 성공 | 실패 |
| 거절한 제안으로 요청 | 요청 성공 | 실패 |
| 관계 없이 요청 | 요청 성공 (기본값) | 실패 |
| 요청 후 관계가 무효해짐 | 이행 및 알림 발송 | 만료, 알림 없음 |
| 프로필 생성이 롤백됨 | 알림만 발송 | 이벤트 미처리 |

---

## 7. 마무리

**되돌릴 수 없는 작업은 커밋 이후로 미룬다.**

알림처럼 외부로 나가는 작업은 트랜잭션과 함께 롤백되지 않습니다. 커밋이 확정된 뒤에만 실행되도록 `AFTER_COMMIT`으로 옮기면, 존재하지 않는 데이터에 대한 알림이 나가는 상황을 구조적으로 막을 수 있습니다.

**커밋 이후의 저장은 새 트랜잭션 경계가 필요하다.**

`AFTER_COMMIT` 시점에는 원래 트랜잭션이 이미 종료돼 있습니다. 이 안에서 상태를 갱신하려면 `REQUIRES_NEW`로 자체 경계를 만들어야 커밋됩니다.

**요청과 이행 사이에는 시간이 있다.**

비동기로 이행되는 기능에서는 요청 시점의 검증 결과가 이행 시점까지 유지된다고 보장할 수 없습니다. 두 시점 사이에 상태를 바꿀 수 있는 경로가 있다면 검증 지점도 두 곳이어야 합니다.

**존재와 유효는 다른 조건이다.**

레코드가 남아 있다는 사실이 그 관계가 아직 유효하다는 뜻은 아닙니다. 취소나 거절처럼 관계를 끝내는 상태가 있다면 조회 조건에 상태를 함께 넣어야 합니다.

**검증 순서는 관계의 수명을 따른다.**

여러 근거가 동시에 성립할 때 짧게 끝나는 관계를 먼저 잡으면 요청이 쉽게 만료됩니다. 오래 유지되는 관계부터 확인하는 편이 요청의 생존율을 높입니다.

이번 글에서는 요청 생성과 이행이 분리된 기능에서 이행을 커밋 이후로 옮기고 검증 시점을 두 곳으로 나눈 과정을 정리해보았습니다. 글의 클래스명과 상태명은 예제이지만, 요청을 저장해두고 나중에 조건이 갖춰졌을 때 처리하는 기능이라면 같은 방식으로 검토할 수 있습니다.

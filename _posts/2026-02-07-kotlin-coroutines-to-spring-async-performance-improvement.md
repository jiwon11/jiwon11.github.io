---
title: "OpenAI API 호출 최적화 — 모델 선택부터 WebClient 튜닝까지"
date: 2026-02-07
tags: [spring-boot, kotlin, webclient, reactor-netty, openai, performance]
category: 개발기
readTime: 12min
mermaid: true
image:
  path: "https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1200&q=80"
  thumbnail: "https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=600&q=80"
---

# OpenAI API 호출 최적화 — 모델 선택부터 WebClient 튜닝까지

안녕하세요. 백엔드 개발자 정지원입니다.

이 글에서는 Kotlin과 Spring WebClient로 LLM API를 호출할 때 모델 선택과 네트워크 설정을 어떻게 검증했는지 정리합니다. 특정 서비스의 대화 횟수, 프롬프트, 모델명, 토큰 한도, 연결 수, 타임아웃, 재시도 횟수, 성능 수치는 다루지 않습니다.

> **공개 범위 안내**
>
> 모든 모델 이름은 A·B 같은 가상 명칭으로, 모든 설정은 프로퍼티 이름으로 표현했습니다. 표의 결과도 실제 측정값이 아니라 비교 방법을 설명하기 위한 상대 평가입니다. 현재 또는 과거 운영 설정을 추론할 수 있는 정보는 포함하지 않습니다.
{: .prompt-info }

---

## 1. 최적화 문제를 두 층으로 나누기

LLM 호출이 느리거나 불안정할 때 원인을 한 덩어리로 보면 해결하기 어렵습니다.

~~~mermaid
flowchart LR
    P["LLM 호출 문제"] --> M["모델 계층"]
    P --> N["네트워크 계층"]

    M --> M1["모델 특성"]
    M --> M2["추론 강도"]
    M --> M3["출력 길이"]

    N --> N1["Connection Pool"]
    N --> N2["Timeout"]
    N --> N3["Retry"]
    N --> N4["Buffer·Compression"]
~~~

모델 계층은 응답 생성 자체의 비용을 결정합니다. 네트워크 계층은 연결 수립, 대기, 장애 전파, 응답 수신을 제어합니다. 두 층을 분리해서 측정해야 “모델이 느린 것”과 “호출 방식이 비효율적인 것”을 구분할 수 있습니다.

---

## 2. 모델은 실제 작업으로 비교한다

벤치마크는 일반 질문이 아니라 애플리케이션이 실제로 수행하는 작업과 비슷한 입력으로 진행해야 합니다. 다만 공개 글에서는 입력 내용과 운영 모델을 노출하지 않고 평가 항목만 남겼습니다.

| 평가 항목 | 확인할 내용 |
|---|---|
| 품질 | 필수 정보 누락, 형식 위반, 사실성 |
| 지연 | 평균보다 p50·p95·최댓값 |
| 비용 | 입력·출력 토큰과 호출 빈도 |
| 안정성 | 빈 응답, 파싱 실패, 일시 오류 |
| 일관성 | 같은 유형 입력에서 결과 편차 |

비교 결과는 다음처럼 상대적으로 기록할 수 있습니다.

| 후보 | 품질 | 지연 | 비용 | 판단 |
|---|---|---|---|---|
| Model A | 기준 | 기준 | 기준 | 비교 기준 |
| Model B | 동등 | 더 짧음 | 더 낮음 | 후보 |
| Model C | 더 높음 | 더 김 | 더 높음 | 복잡한 작업용 |

이 표는 실제 운영 결과가 아닙니다. 중요한 것은 하나의 숫자로 모델을 선택하지 않고 품질·지연·비용·안정성을 함께 비교하는 방식입니다.

### 추론 강도는 작업 난이도에 맞춘다

추론 옵션을 지원하는 모델이라면 복잡한 논리 작업과 단순 변환 작업을 같은 설정으로 호출할 필요가 없습니다.

~~~kotlin
data class ModelPolicy(
    val modelAlias: String,
    val reasoningLevel: String?,
    val maxOutputTokens: Int,
)

fun createRequest(
    input: List<Message>,
    policy: ModelPolicy,
): CompletionRequest =
    CompletionRequest(
        model = policy.modelAlias,
        messages = input,
        reasoningEffort = policy.reasoningLevel,
        maxCompletionTokens = policy.maxOutputTokens,
    )
~~~

구체적인 별칭과 값은 외부 설정이나 모델 정책 저장소에서 주입합니다. 코드 리뷰에서는 값보다 “어떤 작업이 어떤 정책을 선택하는가”와 “변경 시 품질 검증이 있는가”를 확인합니다.

---

## 3. Connection Pool을 직접 설계하기

HTTPS 요청마다 새 연결을 만들면 TCP와 TLS 연결 수립 비용을 반복합니다. Reactor Netty의 <code>ConnectionProvider</code>를 사용하면 연결을 재사용할 수 있습니다.

~~~kotlin
val connectionProvider =
    ConnectionProvider.builder("llm-pool")
        .maxConnections(network.poolCapacity)
        .maxIdleTime(network.idleTimeout)
        .maxLifeTime(network.connectionLifetime)
        .pendingAcquireTimeout(network.acquireTimeout)
        .evictInBackground(network.evictionInterval)
        .build()
~~~

각 값은 다음 근거로 정합니다.

| 설정 | 결정 근거 |
|---|---|
| Pool capacity | 동시 요청량, 외부 API 한도, 인스턴스 자원 |
| Idle timeout | 서버 keep-alive와 유휴 연결 오류 |
| Connection lifetime | 오래된 연결의 실패율 |
| Acquire timeout | 사용자 요청의 전체 시간 예산 |
| Eviction interval | 유휴 연결 정리 비용 |

실제 값은 공개하지 않습니다. 트래픽과 실행 환경이 다르면 같은 값이 오히려 장애를 만들 수 있기 때문입니다.

---

## 4. 타임아웃은 계층별로 둔다

“요청 타임아웃 하나”만 두면 연결, 쓰기, 응답 대기, 읽기 중 어디서 멈췄는지 구분하기 어렵습니다.

~~~mermaid
flowchart LR
    A["Connect"] --> B["Write"]
    B --> C["Response"]
    C --> D["Read"]
~~~

~~~kotlin
val httpClient =
    HttpClient.create(connectionProvider)
        .option(
            ChannelOption.CONNECT_TIMEOUT_MILLIS,
            network.connectTimeoutMillis,
        )
        .responseTimeout(network.responseTimeout)
        .doOnConnected { connection ->
            connection.addHandlerLast(
                ReadTimeoutHandler(network.readTimeoutSeconds),
            )
            connection.addHandlerLast(
                WriteTimeoutHandler(network.writeTimeoutSeconds),
            )
        }
        .compress(true)
~~~

타임아웃 값은 모델별 지연 분포와 사용자 요청의 시간 예산을 기준으로 결정합니다. 너무 짧으면 정상 응답을 실패로 만들고, 너무 길면 느린 외부 호출이 서버 자원을 오래 점유합니다.

---

## 5. 버퍼와 압축은 상한을 전제로 한다

LLM 응답은 일반 JSON API보다 클 수 있습니다. WebClient의 메모리 버퍼는 예상 가능한 최대 응답과 동시 요청 수를 함께 고려해야 합니다.

~~~kotlin
val strategies =
    ExchangeStrategies.builder()
        .codecs { codecs ->
            codecs.defaultCodecs()
                .maxInMemorySize(network.maxInMemoryBytes)
        }
        .build()
~~~

버퍼를 크게 만들면 모든 문제가 해결되는 것은 아닙니다. 동시 요청 수와 곱해지면 메모리 사용량이 커지므로 다음을 함께 확인해야 합니다.

- 출력 토큰 상한
- 응답 스트리밍 가능 여부
- 동시 요청량
- 역직렬화 전 원문 보관 여부
- 초과 응답의 실패 처리

gzip은 텍스트 응답의 전송량을 줄일 수 있지만 실제 효과는 응답 크기와 네트워크 환경을 측정해 판단해야 합니다.

---

## 6. 재시도는 오류 종류와 시간 예산으로 제한한다

재시도는 일시 오류에만 적용해야 합니다. 잘못된 요청이나 인증 실패처럼 같은 입력으로 성공할 가능성이 없는 오류를 반복하면 외부 API와 서버 모두에 부담을 줍니다.

~~~kotlin
fun retrySpec(policy: RetryPolicy): Retry =
    Retry.backoff(policy.maxAttempts, policy.initialBackoff)
        .maxBackoff(policy.maxBackoff)
        .jitter(policy.jitter)
        .filter(::isTransientFailure)
~~~

~~~mermaid
flowchart TD
    A["호출 실패"] --> B{"일시 오류인가?"}
    B -->|No| C["즉시 실패 변환"]
    B -->|Yes| D{"시간 예산이 남았는가?"}
    D -->|No| C
    D -->|Yes| E["Backoff + Jitter"]
    E --> F["재시도"]
~~~

구체적인 오류 목록과 횟수는 외부 API 계약과 사용자 SLA에 따라 달라집니다. 공개 글에서는 실제 필터와 재시도 값을 제외했습니다.

### Thundering Herd 방지

여러 요청이 동시에 실패한 뒤 같은 간격으로 재시도하면 외부 API에 다시 부하가 집중됩니다. 지수 백오프에 jitter를 추가하면 재시도 시점을 분산할 수 있습니다.

---

## 7. 외부 API 예외를 애플리케이션 경계에서 변환한다

WebClient 예외를 코어 로직까지 그대로 전달하면 비즈니스 계층이 HTTP와 Reactor 구현에 결합됩니다.

~~~kotlin
sealed interface LlmFailure {
    data object TimedOut : LlmFailure
    data object RateLimited : LlmFailure
    data object InvalidResponse : LlmFailure
    data class Unavailable(val cause: Throwable) : LlmFailure
}
~~~

어댑터가 외부 오류를 제한된 실패 타입으로 바꾸면 상위 계층은 다음 행동만 결정하면 됩니다.

- 사용자에게 재시도 가능 여부를 안내할지
- 대체 경로를 사용할지
- 비동기 보정 대상으로 남길지
- 운영 알림을 보낼지

이 행동 정책 역시 서비스 내부 규칙이므로 이 글에서는 특정 선택을 설명하지 않습니다.

---

## 8. 관측 데이터는 내용보다 구조가 중요하다

LLM 호출 로그에는 최소한 다음과 같은 기술 지표가 필요합니다.

| 필드 종류 | 예시 |
|---|---|
| 작업 구분 | 공개용 별칭 |
| 모델 구분 | 공개용 모델 그룹 |
| 시간 | 전체 지연, 재시도 지연 |
| 사용량 | 입력·출력 토큰 |
| 결과 | 성공, 시간 초과, 제한, 파싱 실패 |

프롬프트와 응답 원문은 개인정보와 내부 정책을 포함할 수 있으므로 기본 수집 대상으로 두지 않습니다. 필요하다면 마스킹, 샘플링, 접근 권한, 보존 기간을 함께 설계해야 합니다.

---

## 9. Hexagonal Architecture에서 외부 호출 격리하기

~~~kotlin
interface LlmClientPort {
    fun complete(
        messages: List<Message>,
        policyKey: String,
    ): LlmResult
}
~~~

코어는 WebClient, 커넥션 풀, 모델 공급자, 재시도 구현을 알지 않습니다. 인프라 어댑터가 정책 키를 외부 API 요청으로 변환합니다.

~~~mermaid
flowchart LR
    U["Use Case"] --> P["LLM Client Port"]
    A["WebClient Adapter"] -. implements .-> P
    A --> E["External LLM API"]
~~~

이 경계를 두면 모델 공급자나 네트워크 라이브러리를 바꿔도 코어 로직의 변경을 줄일 수 있고, 테스트에서는 포트를 대역으로 교체할 수 있습니다.

---

## 10. 결론

LLM API 최적화는 특정 모델이나 숫자를 복사하는 작업이 아닙니다.

1. 실제 작업과 비슷한 입력으로 모델을 비교한다.
2. 품질·지연·비용·안정성을 함께 본다.
3. 연결 풀과 타임아웃을 실행 환경에 맞춰 외부화한다.
4. 재시도는 일시 오류와 전체 시간 예산 안에서만 허용한다.
5. 외부 API 예외와 구현 세부사항을 어댑터 경계에 가둔다.
6. 프롬프트와 운영 정책이 관측 데이터로 새지 않도록 한다.

이 글의 코드와 표는 설계 방법을 설명하기 위한 예시입니다. 어떤 모델과 값을 실제로 사용하는지는 공개 범위에서 제외했습니다.

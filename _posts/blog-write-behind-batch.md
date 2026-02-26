# 만보기 앱에서 Write-Behind 배치 패턴 적용기 — 걸음수 데이터의 멱등성과 정합성 사이에서

> App Store 건강 카테고리 1위를 달성한 소셜 만보기 앱 "찰리와 걷기"에서 걸음수 기록 시스템을 설계하며 겪은 기술적 의사결정과 트레이드오프를 정리한 글입니다.

## 목차

1. [문제 정의: 만보기 앱의 데이터 특성](#1-문제-정의-만보기-앱의-데이터-특성)
2. [Write-Behind 패턴 선택과 배치 전송 구조](#2-write-behind-패턴-선택과-배치-전송-구조)
3. [Upsert로 멱등성 확보하기](#3-upsert로-멱등성-확보하기)
4. [건강 레벨 계산 — 배치 내 연쇄 의존성](#4-건강-레벨-계산--배치-내-연쇄-의존성)
5. [증분 vs 전체 재계산 — 정확성을 선택한 이유](#5-증분-vs-전체-재계산--정확성을-선택한-이유)
6. [Cache-Aside로 읽기 경로 최적화](#6-cache-aside로-읽기-경로-최적화)
7. [Cron 스케줄러와 이벤트 기반 후속 처리](#7-cron-스케줄러와-이벤트-기반-후속-처리)
8. [전체 아키텍처와 데이터 흐름](#8-전체-아키텍처와-데이터-흐름)
9. [회고: 지금 다시 설계한다면](#9-회고-지금-다시-설계한다면)

---

## 1. 문제 정의: 만보기 앱의 데이터 특성

만보기 앱의 걸음수 데이터는 일반적인 CRUD와는 다른 독특한 특성을 가진다.

**초 단위로 변하는 데이터:**
사용자가 걷는 동안 걸음수는 실시간으로 증가한다. 하루 평균 8,000~10,000보를 걷는 사용자라면, 매 걸음마다 서버에 기록하는 것은 수천 건의 API 요청을 의미한다.

**불안정한 네트워크 환경:**
모바일 앱이기 때문에 지하철, 터널, 엘리베이터 등 네트워크가 끊기는 상황이 빈번하다. 데이터 전송 실패 시 재시도가 필수적이다.

**단방향 증가 데이터:**
걸음수는 하루 동안 오직 증가만 한다. 만약 서버에 기록된 걸음수가 줄어든다면 사용자 입장에서는 "내 걸음수가 사라졌다"는 치명적인 버그로 인식된다.

이 세 가지 특성을 동시에 만족시키려면, 매 걸음마다 실시간으로 기록하는 방식이 아닌 **다른 접근이 필요**했다.

---

## 2. Write-Behind 패턴 선택과 배치 전송 구조

Write-Behind는 캐시 전략에서 유래한 패턴으로, 데이터를 즉시 저장소에 쓰지 않고 **로컬에 버퍼링한 뒤 일정 주기로 배치 전송**하는 방식이다. 찰리와 걷기에서는 이를 클라이언트-서버 간 통신에 적용했다.

### 동작 흐름

```
iOS 클라이언트                              NestJS 서버
━━━━━━━━━━━━━                          ━━━━━━━━━━━━
HealthKit에서 걸음수 수집
     │
     v
앱 내부 로컬 저장
(며칠치 데이터 버퍼링)
     │
     │  앱 포그라운드 진입 / 주기적 sync
     │
     v
POST /log                    ───────>   create(CreateLogDto[], userId)
Body: [                                       │
  { date: "10/01", steps: 8500 },            for each dto:
  { date: "10/02", steps: 12000 },             ├── findFirst (기존 기록 확인)
  { date: "10/03", steps: 6300 },              ├── Math.max (큰 값 채택)
  { date: "10/04", steps: 3200 },              ├── 레벨 재계산
]                                              └── prisma.upsert
                                                     │
                                                     v
                                              MySQL Log 테이블
                                              UNIQUE(date, userId)
```

핵심은 클라이언트가 **배열(CreateLogDto[])** 형태로 며칠치 데이터를 한 번에 전송한다는 점이다. 3일 동안 오프라인이었다가 앱을 열면, 3일치 걸음수가 한 번의 API 호출로 서버에 동기화된다.

### API 설계

```typescript
// dto/create-log.dto.ts
export class CreateLogDto {
  readonly formattedDate: string;  // "2022-10-04"
  readonly steps: number;         // 8500
}

// POST /log — 배열 형태로 수신
async create(createLogDto: CreateLogDto[], userId: string): Promise<any>
```

배열의 마지막 항목이 "오늘"이라는 규칙을 활용해, `loggedIn` 플래그도 함께 처리한다:

```typescript
loggedIn: createLogDto.at(-1) === logDto ? true : false,
// 배열의 마지막 항목(= 오늘)만 loggedIn = true
```

---

## 3. Upsert로 멱등성 확보하기

모바일 앱에서 네트워크 재시도는 일상이다. 같은 데이터가 여러 번 전송될 수 있으므로 서버는 이를 안전하게 처리해야 한다.

### 멱등성 보장 메커니즘

DB 스키마에 `UNIQUE(date, userId)` 복합 유니크 제약을 설정하고, Prisma의 `upsert`를 활용했다:

```typescript
// 1) 기존 기록 조회
const currentLogExist = await this.prisma.log.findFirst({
  where: { userId, date: log.date },
});

// 2) 기존 기록이 있으면 → 더 큰 걸음수를 채택
if (currentLogExist) {
  log.steps = Math.max(log.steps, currentLogExist.steps);
}

// 3) Upsert: 있으면 UPDATE, 없으면 INSERT
const createdLog = await this.prisma.log.upsert({
  where: { userDate: { date: log.date, userId } },
  update: { level: log.level, steps: log.steps },
  create: log,
});
```

### 시나리오별 동작

| 시나리오 | 서버 기존값 | 클라이언트 전송값 | Math.max 결과 | DB 동작 |
|:---|:---:|:---:|:---:|:---|
| 최초 전송 | 없음 | 8,500 | 8,500 | INSERT |
| 앱 재시작 후 같은 값 재전송 | 8,500 | 8,500 | 8,500 | UPDATE (변화 없음) |
| 걸음수 증가 후 재전송 | 8,500 | 12,000 | 12,000 | UPDATE (갱신) |
| 오래된 데이터 재전송 | 12,000 | 8,500 | 12,000 | UPDATE (유지) |

`Math.max`를 사용하는 이유는 명확하다. **걸음수는 하루 동안 절대 줄어들지 않는다.** 서버에 이미 12,000보가 기록되어 있는데 클라이언트가 8,500보를 보냈다면, 이는 네트워크 지연이나 앱 상태 불일치로 인한 것이다. 이 경우 더 큰 값(12,000)을 유지하는 것이 올바른 판단이다.

### 왜 단순 INSERT가 아닌 Upsert인가

단순 INSERT + `ON DUPLICATE KEY` 에러 핸들링 대신 Upsert를 택한 이유:

1. **에러를 비즈니스 로직으로 처리하지 않기 위해.** 중복 INSERT 시 에러를 catch하고 UPDATE로 분기하는 것은 예외 흐름을 정상 흐름처럼 사용하는 안티패턴이다.
2. **원자적(Atomic) 연산 보장.** Prisma의 upsert는 내부적으로 단일 트랜잭션으로 처리되어 race condition을 방지한다.
3. **코드 의도가 명확하다.** "있으면 업데이트, 없으면 생성"이라는 의도가 `upsert`라는 메서드 이름 자체에 드러난다.

---

## 4. 건강 레벨 계산 — 배치 내 연쇄 의존성

찰리와 걷기에는 걸음수에 따라 캐릭터의 건강 상태가 5단계로 변하는 게이미피케이션 요소가 있다:

```
veryObese → obese → overweight → healthy → veryHealthy
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
      걸음수 증가 →
```

레벨 계산 규칙은 다음과 같다:

- **레벨 상승:** 당일 걸음수 >= 10,000보 → initLevel에서 +1
- **레벨 하락:** 전일 걸음수 < 6,000보 → 전일 level에서 -1 (다음 날 initLevel에 반영)
- **유지:** 6,000 <= 걸음수 < 10,000 → 변동 없음

```typescript
// 어제 걸음수 기반 초기 레벨 결정
calculateCurrentInitLevel(previousLog: Log): Level {
  if (previousLog.steps < 6000) {
    return this.levelDown(previousLog.level);  // 한 단계 하락
  }
  return previousLog.level;                     // 유지
}

// 오늘 걸음수 기반 현재 레벨 결정
calculateLevel(currentLog: LogEntity): Level {
  if (currentLog.steps >= 10000) {
    return this.levelUp(currentLog.initLevel);  // 한 단계 상승
  }
  return currentLog.initLevel;                   // 유지
}
```

### 왜 for...of 순차 처리인가

배치로 들어온 배열 `[10/01, 10/02, 10/03, 10/04]`에서, 10월 2일의 `initLevel`은 10월 1일의 `level`에 의존하고, 10월 1일의 `level`은 10월 1일의 걸음수에 의존한다.

```
10/01 steps=8500 → level=healthy     (10000 미만이므로 initLevel 유지)
10/02 initLevel = 10/01의 level       → healthy (8500 >= 6000이므로 하락 없음)
10/02 steps=12000 → level=veryHealthy (10000 이상이므로 상승)
10/03 initLevel = 10/02의 level       → veryHealthy
...
```

이 연쇄 의존성 때문에 `Promise.all`이나 `Array.map`으로 병렬 처리할 수 없다. 반드시 `for...of`로 **날짜순 순차 처리**해야 한다.

이것은 단순한 구현 선택이 아니라, 도메인 로직이 강제하는 제약이다. 배치 처리라고 해서 항상 병렬화할 수 있는 것이 아님을 보여주는 사례이기도 하다.

---

## 5. 증분 vs 전체 재계산 — 정확성을 선택한 이유

Write-Behind로 걸음수를 기록한 후, 사용자 통계(누적 걸음수, 주간 걸음수, 최대 일일 걸음수 등)를 어떻게 집계할 것인가? 이 질문에 대해 두 가지 방식을 모두 시도했다.

### Phase 1: 증분(Incremental) 업데이트 — 2022.08

처음에는 성능을 고려해 증분 방식을 채택했다. 이미 계산된 통계값에 새 로그의 차분만 더하는 방식이다:

```typescript
// 주석 처리된 코드 (log.service.ts 270~412행)
async updateUserStats(userId, lastestLog, logs) {
  const prev = await this.prisma.userStats.findFirst({ where: { userId } });
  let { accumSteps, weeklySteps, maxDailySteps } = prev;

  logs.forEach((log) => {
    const sameDateUpdated = moment(lastLoggedInDate).isSame(moment(log.date));
    if (sameDateUpdated) {
      // 같은 날짜의 업데이트 → 차분만 계산
      const stepDiff = log.steps > lastedLogSteps ? log.steps - lastedLogSteps : 0;
      weeklySteps = weeklySteps + stepDiff;
    }
    // ... 7가지 필드를 각각 조건부 업데이트
  });
}
```

이론적으로는 O(1)이라 빠르다. 하지만 실서비스에서 문제가 터졌다.

**엣지 케이스의 폭발:**

| 엣지 케이스 | 증분 방식에서의 문제 |
|:---|:---|
| 같은 날짜에 걸음수가 여러 번 업데이트 | 차분 계산이 누적되며 실제보다 많은 값이 됨 |
| 월요일에 주간 걸음수 리셋 | 리셋 시점 판단 로직이 추가로 필요 |
| 앱 재설치 후 전체 데이터 재전송 | 이전 통계값과 전체 데이터가 이중 합산됨 |
| 서버 마이그레이션 | 기존 통계값이 유실되면 복구 불가 |

7개 통계 필드 각각에 대해 이런 분기 처리를 해야 했고, 하나라도 빠지면 **통계 버그가 발생**했다. 실제로 여러 차례 걸음수 통계 불일치 이슈가 보고되었다.

### Phase 2: 전체 재계산(Full Recalculation) — 현재

증분 방식의 버그를 근본적으로 해결하기 위해, 통계 요청 시 **모든 로그를 처음부터 순회**하는 방식으로 전환했다:

```typescript
async calcUserStats(userId: string): Promise<UserStatsEntity> {
  const thisWeekStartDate = moment().startOf("isoWeek").format("YYYY-MM-DD");
  const userLogs = await this.findByUserId("service", userId, "10000", "0", "asc");

  let accumSteps = 0, maxDailySteps = 0, todaySteps = 0,
      weeklySteps = 0, maxLoginStreak = 0;

  userLogs.forEach((log: Log) => {
    const logDate = new Date(log.date).toJSON().split("T")[0];
    accumSteps += log.steps;
    maxDailySteps = Math.max(maxDailySteps, log.steps);
    maxLoginStreak = log.loggedIn ? maxLoginStreak + 1 : 0;
    if (logDate === moment().format("YYYY-MM-DD")) todaySteps = log.steps;
    if (logDate >= thisWeekStartDate) weeklySteps += log.steps;
  });

  return { accumSteps, maxDailySteps, todaySteps, weeklySteps, ... };
}
```

### 트레이드오프 분석

|  | 증분 방식 | 전체 재계산 |
|:---|:---|:---|
| **시간 복잡도** | O(1) | O(N) |
| **정확성** | 엣지 케이스에서 부정확 | **항상 정확** |
| **코드 복잡도** | 높음 (7개 필드 × 조건부 분기) | **낮음** (단순 합산) |
| **상태 관리** | UserStats 테이블 필요 | 불필요 (매번 계산) |
| **장애 복구** | 통계값 유실 시 복구 어려움 | **Log만 있으면 복구 가능** |

### 왜 이 선택이 합리적이었는가

실서비스 기준으로 계산해보면:

- 1년 활동 사용자: 하루 1건 × 365일 = **365건** 순회
- `forEach` + 단순 합산 = 메모리 내 연산이라 수 ms 이내
- DB 조회가 병목이지만, `UNIQUE(date, userId)` 인덱스 덕분에 인덱스 스캔으로 빠르게 조회
- 실측 API 응답: **200ms 이내**

사용자 수가 적은 스타트업 서비스에서는 O(N) 전체 재계산의 성능 비용보다 **버그로 인한 사용자 이탈 비용이 훨씬 크다.** 정확성과 성능 사이에서 정확성을 선택한 것은 서비스 규모에 맞는 합리적 판단이었다.

> 물론 사용자가 수만 명으로 늘어난다면, DB 레벨 집계(`SELECT SUM(steps)`)나 UserStats 증분 업데이트 + 주기적 정합성 검증 방식으로의 전환이 필요하다. 이 부분은 [9장](#9-회고-지금-다시-설계한다면)에서 다룬다.

---

## 6. Cache-Aside로 읽기 경로 최적화

Write-Behind로 쓰기를 최적화했다면, 읽기는 어떻게 최적화했을까?

### 문제: 하우스 랭킹 조회의 N+1

찰리와 걷기에는 "하우스"라는 그룹 기능이 있다. 사용자들이 팀을 만들어 주간 걸음수를 경쟁한다. 하우스 상세 조회 시 각 멤버의 주간 걸음수를 계산해야 한다:

```typescript
// house.service.ts
async getHousemates(houseId: number, limit?: number): Promise<any> {
  const houseUsers = await this.prisma.userHouse.findMany({
    where: { houseId },
    include: { user: { select: { nickname: true, charlieImageNo: true } } },
  });

  // 각 멤버의 주간 걸음수를 계산 — N+1 문제!
  for (let user of houseUsers) {
    const { weeklySteps, lastLoggedInDate } =
      await this.logService.calcUserStats(user.userId);
    // calcUserStats는 O(로그 수) 순회...
    houseUsers[houseUsers.indexOf(user)]["user"]["UserStats"] =
      { weeklySteps, lastLoggedInDate };
  }

  // 주간 걸음수 기준 내림차순 정렬
  houseUsers.sort((a, b) =>
    b["user"]["UserStats"]["weeklySteps"] -
    a["user"]["UserStats"]["weeklySteps"]
  );

  return limit ? houseUsers.slice(0, limit) : houseUsers;
}
```

멤버가 10명이고 각각 365건의 로그가 있다면, 한 번의 랭킹 조회에 **3,650건의 로그를 순회**해야 한다. 피크 시간에 여러 하우스가 동시에 조회되면 DB 부하가 급증한다.

### 해결: Redis Cache-Aside

하우스 멤버 랭킹 데이터를 Redis에 캐싱하여, Cache Hit 시 DB 쿼리 없이 즉시 반환한다:

```typescript
// house.controller.ts — Cache-Aside 패턴
async getHouseInfo(houseId: number) {
  const cacheExists = await this.cacheService.existHouse(houseId);

  if (cacheExists) {
    // Cache Hit → Redis에서 즉시 반환 (weeklySteps 내림차순 정렬 포함)
    return await this.cacheService.getHouseUsers(houseId);
  } else {
    // Cache Miss → DB 조회 → 통계 계산 → Redis 적재 → 반환
    const houseDetails = await this.houseService.getHouseInfo(houseId);
    await this.cacheService.initHouseUsers(houseId, houseDetails.users);
    return houseDetails;
  }
}
```

Redis OM(Object Mapping)을 사용하여, Redis 내에서 `weeklySteps` 기준 정렬까지 처리한다:

```typescript
// cache.service.ts — Redis OM 기반 캐시 서비스
async getHouseUsers(houseId: number): Promise<any> {
  const users = await this.houseUserRepository.search()
    .where("houseId").eq(houseId)
    .sortDescending("weeklySteps")  // Redis 레벨에서 정렬
    .return.all();

  return users.map((user, idx) => ({
    ...user.toJSON(),
    rank: idx + 1,  // 순위 부여
  }));
}
```

### 캐시 전략의 한계와 트레이드오프

이 구현에는 명시적인 **캐시 무효화(invalidation) 로직이 없다.** 걸음수가 업데이트되어도 Redis의 랭킹 데이터는 갱신되지 않는다. 이는 의도적인 설계인데:

- 하우스 랭킹은 **실시간 정확성이 필수적이지 않다.** 몇 분~수십 분의 지연은 허용 가능하다.
- 캐시가 만료(Redis TTL)되거나 서비스가 재시작되면 자연스럽게 최신 데이터로 갱신된다.
- 복잡한 캐시 무효화 로직을 추가하면, 오히려 캐싱의 이점이 줄어든다.

---

## 7. Cron 스케줄러와 이벤트 기반 후속 처리

Write-Behind로 쌓인 걸음수 데이터는 매주 집계되어 메달 수여에 활용된다.

### 주간 메달 수여 스케줄러

```typescript
// task.service.ts — @nestjs/schedule 기반
@Cron("59 23 * * sun", {
  name: "awardMedalsTask",
  timeZone: "Asia/Seoul",       // 한국 시간대 명시
})
async awardMedals() {
  this.logger.log(`run job ${this.awardMedals.name}!`);
  this.eventEmitter.emit("medals.awarded");  // 이벤트 발행
}
```

매주 일요일 23:59(KST), 각 하우스별 주간 걸음수 상위 3명에게 메달을 자동 수여한다:

```typescript
// medal.service.ts
async awardMedals(filterDateDto: FilterDateDto): Promise<any> {
  const populatedHouses = await this.prisma.userHouse.groupBy({ by: ["houseId"] });
  const topN = Object.keys(MedalColor).length; // gold, silver, bronze = 3

  for (const house of populatedHouses) {
    const winners = await this.houseService.getHousemates(house.houseId, topN);
    for (const i in winners) {
      await this.prisma.userHouseMedal.create({
        data: {
          userHouseId: winners[i].id,
          medalColor: Object.keys(MedalColor)[i],  // 0→gold, 1→silver, 2→bronze
        },
      });
    }
  }
}
```

### 왜 EventEmitter인가

`medalService.awardMedals()`를 직접 호출하지 않고 `eventEmitter.emit("medals.awarded")`를 사용한 이유:

1. **모듈 간 결합도 감소:** TaskService가 MedalService에 직접 의존하지 않는다.
2. **후속 작업 확장성:** 메달 수여 후 알림 전송, 캐시 갱신 등을 이벤트 리스너 추가만으로 확장할 수 있다.
3. **관심사 분리:** "언제 실행할 것인가(스케줄)"와 "무엇을 할 것인가(메달 수여)"를 분리한다.

---

## 8. 전체 아키텍처와 데이터 흐름

지금까지 설명한 내용을 하나의 그림으로 정리하면:

```
┌─────────────────────────────────────────────────────────────┐
│                    쓰기 경로 (Write-Behind)                   │
│                                                             │
│  HealthKit → 앱 로컬 버퍼 → POST /log (배열)                 │
│                                │                            │
│                                v                            │
│                     for each dto (순차):                     │
│                       findFirst → Math.max → upsert         │
│                                │                            │
│                                v                            │
│                        MySQL Log 테이블                      │
│                      UNIQUE(date, userId)                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    읽기 경로 (Cache-Aside)                    │
│                                                             │
│  GET /house/:id → Redis 확인                                │
│                     │                                       │
│              ┌──────┴──────┐                                │
│              │             │                                │
│           Hit │          Miss │                              │
│              v             v                                │
│         Redis 반환    MySQL 조회                             │
│     (weeklySteps      → calcUserStats(O(N))                 │
│      내림차순 정렬)     → Redis 적재                         │
│                          → 반환                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    집계 경로 (Cron Batch)                     │
│                                                             │
│  매주 일 23:59 KST                                          │
│    → eventEmitter.emit("medals.awarded")                    │
│    → 각 House별 getHousemates(topN=3)                       │
│    → calcUserStats × 멤버 수                                │
│    → 상위 3명 메달 수여 (gold/silver/bronze)                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. 회고: 지금 다시 설계한다면

8개월간 실서비스를 운영하며 이 구조의 장단점을 체감했다. 다시 설계한다면 개선하고 싶은 부분들이 있다.

### 1) calcUserStats의 O(N) 순회 제거

현재 방식은 정확하지만, 서비스가 성장하면 병목이 된다. DB 레벨 집계로 대체할 수 있다:

```sql
-- 현재: 애플리케이션 레벨에서 전체 순회
SELECT * FROM Log WHERE userId = ? ORDER BY date ASC LIMIT 10000;
-- → 앱에서 forEach로 합산

-- 개선: DB 레벨 집계
SELECT
  SUM(steps) AS accumSteps,
  MAX(steps) AS maxDailySteps,
  SUM(CASE WHEN date >= ? THEN steps ELSE 0 END) AS weeklySteps
FROM Log
WHERE userId = ?;
```

### 2) 증분 + 정합성 검증 하이브리드

증분 업데이트의 성능과 전체 재계산의 정확성을 모두 취하는 방법:

```
실시간: 증분 업데이트 (O(1))
    └─ 빠른 응답

새벽 3시: Cron으로 전체 재계산 → 증분 결과와 비교
    └─ 불일치 발견 시 보정 + 알림
```

### 3) Event Sourcing 도입

걸음수 기록을 불변 이벤트 로그로 관리하면:

- 현재의 `Math.max` + Upsert 대신, 모든 변경 이력을 보존
- 시점 복원(point-in-time recovery) 가능
- 다만 스타트업 초기에는 오버엔지니어링이 될 수 있다

### 4) 캐시 무효화 전략 추가

걸음수 업데이트 시 해당 하우스의 Redis 캐시를 명시적으로 무효화하면, 실시간에 더 가까운 랭킹을 제공할 수 있다:

```typescript
// log.service.ts create() 끝에 추가
const userHouses = await this.prisma.userHouse.findMany({ where: { userId } });
for (const uh of userHouses) {
  await this.cacheService.invalidateHouse(uh.houseId);
}
```

---

## 마무리

Write-Behind 패턴은 "모든 데이터를 즉시 저장한다"는 전통적 접근에서 벗어나, **"데이터의 특성에 맞게 쓰기 시점을 조절한다"**는 사고의 전환을 보여준다.

만보기 앱이라는 구체적인 도메인에서:

- **Write-Behind + Upsert**로 네트워크 불안정 환경에서의 데이터 무결성을 확보했고,
- **증분 → 전체 재계산**으로 정확성과 유지보수성을 택했으며,
- **Cache-Aside**로 읽기 성능을 보완하고,
- **Cron + EventEmitter**로 주기적 집계와 느슨한 결합을 달성했다.

완벽한 설계는 아니었지만, 서비스 규모와 팀 리소스에 맞는 **실용적인 선택**이었다. 그리고 그 과정에서 "정확성 vs 성능"이라는 영원한 트레이드오프를 실무에서 직접 겪고 판단한 것이 가장 값진 경험이었다.

---

**기술 스택:** NestJS, Prisma, MySQL, Redis (ioredis + Redis OM), @nestjs/schedule, Docker, AWS ECS
**서비스:** 찰리와 걷기 — App Store 건강 카테고리 1위 (2022.06 ~ 2023.02)

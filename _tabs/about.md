---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

## 👋 안녕하세요!

백엔드 개발자 **NOWIL**입니다.

주로 **Spring Boot**, **Kotlin**, **AWS** 기반의 백엔드 시스템을 개발하고 있으며,
이 블로그에서는 개발 과정에서 겪은 문제 해결 경험과 학습 내용을 공유합니다.

## 🛠️ Tech Stack

- **Backend**: Spring Boot, Kotlin, Node.js
- **Database**: PostgreSQL, MongoDB, Redis
- **Cloud**: AWS (ECS, Lambda, RDS, S3)
- **DevOps**: Docker, GitHub Actions

## 💼 프로젝트 경력

### Duurian 백엔드 담당 (2025.11 ~)

**개인 맞춤형 만남 제공 매칭 서비스 'Duurian' 백엔드 개발**

`Restful API 구현` `요구사항 분석 및 데이터 모델링` `AI 서버 연동 서비스 구현`

- **프로젝트 개요**
  - Spring 및 PostgreSQL 기반 서버측 애플리케이션 구현
  - 기술 키워드: Kotlin, Spring, JPA, PostgreSQL, OpenAI, Async
  - 아키텍처: Hexagonal + Multi-module (api, core, domain, infrastructure)
- **담당 업무**
  - 모듈 분리를 통해 웹 어댑터, 유스케이스, 도메인 모델, 영속성 어댑터를 분리
  - 듀리와의 대화(OpenAI ChatGPT API 연동을 통한 채팅 후 보상 지급) 통합 구현
  - 대화/요약/페르소나/프롬프트/리워드 등 **핵심 도메인 수직 통합 구현**
  - **대화 처리 → 요약 → 리워드(보상) 지급**으로 이어지는 **후처리 체인**을 설계·구현
- **주요 기능 개발**
  - 외부 LLM(OpenAI) 연동 안정화 및 요청 구조 표준화
  - 대화 요약 생성의 **비동기 처리 구조** 확립 (**71% 성능 개선** 달성)
  - 대화·질문·요약·프롬프트 기능을 **엔드투엔드로 설계/구현**
  - 리워드(보상) 지급/조회 로직을 정리해 **도메인 확장성** 강화
  - 페르소나 공개 여부 관련 버그 수정으로 **데이터 신뢰성 개선**
  - 비동기 구조로 **응답 지연 최소화** 및 **DIP 기반 확장성 확보**

### Emerdy 백엔드 담당 (2023.02 - 2023.12)

**딥러닝과 음성 처리를 통한 위급 상황 대응 앱 'Emerdy' 백엔드 개발**

`아키텍처·인프라 설계` `요구사항 분석 및 데이터 모델링 총괄` `AI 서버 연동 서비스 구현`

- **프로젝트 개요**
  - Nest.js 및 MySQL 기반 서버측 애플리케이션 구현
- **담당 업무**
  - 백엔드 시스템 설계 및 구현
  - 데이터베이스 모델링 설계 및 구현
  - Restful API 구현
  - 음성 처리 파이프라인 구축
  - 양방향 실시간 사용자 위치 데이터 통신
  - 인프라 및 배포 환경 구축
- **시스템 설계**
  - 요구사항을 파악하여 MySQL 데이터 모델링 및 구현
- **실사용 테스트**
  - 50명의 잠재 고객을 대상으로 음성 녹음 시작 동작 트리거에 대한 사용자 테스트
  - 사용자 피드백 기반 기능 개선 주도
- **주요 기능 개발**
  - GeoJSON 형식으로 위치 데이터 저장
  - FFMPEG를 통해 음성 데이터 처리 최적화로 음성 데이터 분류 시간 3초 단축
  - 음성 처리 파이프라인 구축 (S3 → AWS Transcribe → ML 서버 → DB)
  - Socket.io를 통한 양방향 실시간 사용자 위치 데이터 통신 구현
  - Message Queue인 Bull.js를 통해 비동기 Push Notification 구현하여 작업 분리로 서버 부하 최소화
- **인프라 및 배포**
  - Naver Clova → AWS Transcribe로 전환하여 음성 데이터 처리 최적화를 통해 위급 상황 분류 알고리즘 정확도 향상
  - Docker, Docker-compose, GitHub Actions 기반 AWS 인프라 운영
  - Nginx Reverse Proxy를 통한 로드 밸런싱 및 SSL 처리

### Devigation 백엔드 담당 (2023.08 - 2023.11)

**개발자 커리큘럼 공유 웹 플랫폼 'Devigation' 백엔드 개발**

`아키텍처 설계` `요구사항 분석 및 데이터 모델링 총괄` `Restful API 개발` `테스트 코드 표준화 및 코드 리뷰 문화 정착`

- **프로젝트 개요**
  - Spring 2.7(SpringBoot) 및 MySQL 기반 서버 애플리케이션 구현
- **담당 업무**
  - DDD 기반 도메인 레이어 아키텍처 설계
  - 인증 및 보안 도메인 설계 및 구현
  - 데이터베이스 모델링 설계 및 구현
  - Restful API 구현
  - 에러 처리 도메인 설계 및 구현
  - 통합 테스트 환경 구축
- **인증 및 보안**
  - Spring Security를 통한 소셜 로그인 구현
  - JWT 기반 인증 및 인가 시스템을 개발하여 Stateless 인증으로 확장성 확보
  - OAuth2(Github) 로그인 지원 구현
- **시스템 설계**
  - 객체 지향 기반의 도메인 레이어 설계 및 개발
  - DDD 원칙으로 비즈니스 로직 응집도 향상
  - 요구사항을 파악하여 MySQL 데이터 모델링 및 구현
  - RestControllerAdvice를 통한 에러 응답 코드 핸들링
- **품질 관리**
  - JUnit을 통한 테스트 코드 작성하여 시스템의 안정성과 요구사항 만족 여부를 확인
  - GitHub Actions를 통한 CI/CD 구현하여 개발과 배포 과정의 불확실성과 비효율을 해결
  - Swagger를 통한 모든 엔드포인트 문서화하여 프론트엔드 개발자와 협업 강화

### 찰리와 걷기 백엔드 담당 (2022.08 - 2023.02)

**캐릭터 육성을 통한 소셜 만보기 앱 'Charlie' 백엔드 개발**

`아키텍처·인프라 설계` `요구사항 분석 및 데이터 모델링 총괄` `Restful API 개발` `인프라 구성`

- **프로젝트 개요**
  - Nest.js 및 MySQL 기반 서버측 애플리케이션 구현
- **시스템 설계**
  - 요구사항을 파악하여 MySQL 데이터 모델링 및 구현
- **알고리즘 개발**
  - 일간 걸음 수를 통한 비만도 레벨 계산 알고리즘 개발
- **인프라 및 배포**
  - Docker, Docker-compose, GitHub Actions 기반 AWS 인프라 운영
  - AWS ECS 기반 자동화 CI/CD 파이프라인 구축으로 배포 소요 시간 10분 내로 단축
- **주요 성과**
  - **App Store 건강 및 피트니스 카테고리 무료 인기 앱 1위 달성**
  - App Store 전체 인기 무료 앱 43위 달성
  - **(2022.12 기준) 월간 활성 사용자 2,000명 이상 확보**

### 모브 백엔드 담당 (2020.11 - 2022.05)

**여성 월경 주기 트래킹 및 호르몬 사이클 기반 다이어트 앱 'Mauve' 백엔드 개발**

`아키텍처·인프라 설계` `요구사항 분석 및 데이터 모델링` `Restful API 개발`

- **프로젝트 개요**
  - Express.js 및 MongoDB 기반 서버측 애플리케이션 구현
- **시스템 설계**
  - 요구사항을 파악하여 MongoDB 데이터 모델링 및 구현
- **알고리즘 개발**
  - 사용자의 월경 시작일 및 종료일 데이터 기반 월경 주기 및 호르몬 사이클 예측 알고리즘 개발
- **실시간 통신**
  - socket.io를 통한 채팅 시스템 개발
- **비동기 처리**
  - Message Queue 라이브러리 Bull.js를 통한 비동기 Push Notification 구현
- **배포 및 인프라**
  - ECS의 Dynamic Port Mapping을 이용한 Blue/Green 무중단 배포
  - Docker, Docker-compose, GitHub Actions를 통한 AWS ECS 자동화 CI/CD 파이프라인 구축

## 📫 Contact

- **GitHub**: [jiwon11](https://github.com/jiwon11)
- **Twitter**: [@N0W1L](https://twitter.com/N0W1L)
- **Email**: jeewonre@naver.com

---

> 이 블로그의 모든 글은 개인적인 학습과 경험을 바탕으로 작성되었습니다.

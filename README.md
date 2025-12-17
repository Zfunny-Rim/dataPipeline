## 1. 프로젝트 목적

 - 공공데이터를 단순 수집하는 것이 아니라
의미 있는 파생 데이터(체감날씨) 를 생성하는 데이터 파이프라인을 구현한다.
 - Job 기반 실행, 메시지 큐, 재처리, 상태 관리 등 운영 관점 구조를 학습한다.
 - 단일 서버에서 시작하되, 분산 시스템으로 확장 가능한 구조를 갖춘다.

## 2. 주제 확정

- 기상청 공공데이터포털 – 초단기 실황 조회 API 사용
- 단순 “기온 조회”가 아니라
초단기 실황 데이터를 기반으로 체감온도(feels-like)를 산출하는 시스템

체감온도 산출 규칙:
 - 고온 구간 → Heat Index (기온 + 습도)
 - 저온 구간 → Wind Chill (기온 + 풍속)
 - 온화 구간 → 실제 기온 사용
 - 산출 결과에는 feels_like_c 와 method(HEAT_INDEX | WIND_CHILL | NONE) 저장

## 3. 전체 아키텍처

 - API 서버: Spring Boot
   - 역할: Job 생성, 상태 조회, 재처리 트리거
   - 실제 데이터 수집·계산은 하지 않음 (컨트롤 플레인)
   
 - Worker 서버: Spring Boot (별도 애플리케이션)
   - RabbitMQ 메시지 소비
   - 기상 API 호출
   - Raw 저장 → 정규화 → 체감 산출
   - Job 상태 전이 관리

 - 메시지 브로커: RabbitMQ
   - 비동기 Job 처리
   - DLQ(Dead Letter Queue) 사용

 - DB: PostgreSQL
   - Job 상태
   - Raw 데이터
   - Normalized 데이터
   - Derived(체감) 데이터 저장

 - 스케줄러: Worker 서버 내부 @Scheduled
   - 1시간 주기로 Job enqueue
 
## 4. 데이터 흐름

1. 스케줄러 또는 API 호출로 Job 생성
2. Job을 DB에 QUEUED 상태로 저장
3. RabbitMQ에 Job 메시지 발행
4. Worker가 메시지 소비
5. Job 상태 RUNNING 변경
6. 기상청 초단기 실황 API 호출
7. Raw 데이터 저장(JSON)
8. 정규화 테이블 upsert (멱등성)
9. 체감온도 계산 후 Derived 테이블 저장
10. 성공 시 DONE, 실패 시 FAILED

## 5. Job 설계

- COLLECT_WEATHER
  - 초단기 실황 데이터 수집
  - Raw + Normalized 저장
- CALC_FEELS_LIKE
  - 정규화 데이터를 기반으로 체감온도 산출
  - Derived 테이블 저장

## 6. 데이터 모델 요약

- jobs
  - job_id, job_type, status
  - parameters(JSON)
  - retry_count, error_message
  - created_at, started_at, finished_at
- weather_raw
  - raw_id, location_id
  - payload(JSON), collected_at
- weather_observation (정규화)
  - location_id, observed_at
  - temperature, humidity, wind_speed, precipitation
  - UNIQUE(location_id, observed_at)
- feels_like (파생)
  - location_id, observed_at
  - feels_like_c, method
  - UNIQUE(location_id, observed_at)

## 7. 제공 API

- POST /jobs
- GET /jobs/{id}
- GET /jobs
- GET /weather/observations
- GET /weather/feels-like

## 8. 기술 스택
- Language: Java
- Framework: Spring Boot
- Message Queue: RabbitMQ
- Database: PostgreSQL
- Scheduler: Spring @Scheduled
- Infra: Docker Compose

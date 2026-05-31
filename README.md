# daedongje-yonsei-server

연세대학교 개교 141주년 **무악대동제 2026** 공식 앱의 백엔드 서버.

방문자용 앱과 운영진 어드민을 하나의 Spring Boot 서버로 제공한다. 부스·예약, 공연·라이브 무대, 공지·유실물, 홈·지도, 운영 모니터링까지 축제 기간 동안 필요한 도메인을 담는다.

---

## 프로젝트 소개

축제 방문자는 앱에서 부스를 둘러보고 예약하며, 공연 타임테이블과 현재 진행 중인 무대를 확인한다. 운영진은 어드민 API로 부스·공연·공지를 관리하고, 시스템 상태를 모니터링한다.

**핵심 기능**

- **부스 · 예약** — 일자/구역/검색 기반 부스 조회, 메뉴·이미지, 클릭 로그 기반 인기 부스, 전화번호 기반 예약 (광클 멱등 처리)
- **공연** — 공연 정보·타임테이블, 무대별 라이브 판정(수동 핀 + 시간 자동), 세트리스트, 응원 메시지
- **정보** — 공지사항, 유실물, 배리어프리(장애인 편의) 정보, 만족도 조사
- **홈 · 지도** — 메인 배너, 인기 부스(캐시), 축제장 위치 정보
- **모니터링** — 헬스 체크, 최근 에러 로그, Prometheus 메트릭, Grafana 알림 웹훅

---

## 기술 스택

| 구분 | 사용 기술 |
| --- | --- |
| 언어 / 런타임 | Java 17 (Temurin) |
| 프레임워크 | Spring Boot 3.5.13 (Web, Validation, Data JPA, Cache, Actuator) |
| 데이터베이스 | MySQL 8 · JPA/Hibernate · Flyway (스키마 마이그레이션) |
| 세션 / 캐시 | Redis (Spring Session) · Caffeine (인메모리 캐시) |
| 인증 | 세션 기반 + 역할 기반 접근 제어(RBAC) |
| 스토리지 | AWS S3 (이미지, presigned URL) |
| API 문서 | springdoc OpenAPI 2.8.9 (Swagger UI) |
| 관측(Observability) | Actuator · Micrometer/Prometheus · Grafana(Alloy → Loki/Prometheus) |
| 빌드 | Gradle (Java 17 toolchain) |

---

## 아키텍처 개요

```
com.likelion.yonsei.daedongje
├── domain/            # 도메인별 패키지 (controller · service · repository · entity · dto)
│   ├── auth           # 어드민 계정 · 세션 인증 · 역할(RBAC)
│   ├── booth          # 부스 · 메뉴 · 이미지 · 클릭 로그
│   ├── performance    # 공연 · 타임테이블 · 라이브 무대 · 응원
│   ├── reservation    # 부스 예약
│   ├── info           # 공지 · 유실물 · 배리어프리 · 만족도
│   ├── home           # 배너 · 인기 부스
│   ├── map            # 축제장 위치
│   └── monitoring     # 헬스 · 에러 로그 · Grafana 웹훅
├── common/            # 공통 모듈
│   ├── entity         # BaseEntity (createdAt/updatedAt 자동 관리)
│   ├── response       # ApiResponse<T> · PageResponse<T> 표준 응답
│   ├── exception      # 전역 예외 처리 · 에러 코드
│   └── web            # CORS · 클라이언트 IP 해석 등
└── config/            # OpenAPI, 캐시, 세션 등 설정
```

- **사용자 API(`/api/**`)** 와 **어드민 API(`/api/admin/**`)** 를 경로로 분리한다.
- **인증**: 어드민 로그인 시 세션 쿠키(`DDJ_ADMIN_SESSION`)를 발급하고 세션은 Redis에 저장한다. 인터셉터(`AdminRoleInterceptor`)와 `@RequireAdminRole`로 메서드 단위 역할을 검증한다.
- **응답**: 모든 API는 `ApiResponse<T>`(`success`/`data`/`error`) 포맷으로 통일한다.
- **스키마**: JPA `ddl-auto=validate` — 테이블 생성/변경은 Flyway가 전담하고, JPA는 엔티티-스키마 일치성만 검증한다.

---

## 도메인 구조

| 도메인 | 책임 | 핵심 엔티티 |
| --- | --- | --- |
| `auth` | 어드민 계정·세션 인증, 역할 기반 접근 제어 | `AdminUser`, `AdminRole`, `AdminStatus` |
| `booth` | 부스 정보·메뉴·이미지, 클릭 로그 기반 인기 부스 집계 | `Booth`, `Menu`, `BoothImage`, `BoothClickLog` |
| `performance` | 공연 정보·타임테이블, 라이브 무대(수동 핀+시간 자동), 응원 | `Performance`, `PerformanceSetlist`, `PerformanceImage`, `LivePerformance`, `PerformanceCheerMessage` |
| `reservation` | 부스 예약 생성/관리, 전화번호 기반 조회, 광클 멱등 처리 | `Reservation`, `ReservationStatus` |
| `info` | 공지사항·유실물·배리어프리 정보·만족도 조사 | `Notice`, `LostItem`, `BarrierFreeInfo` |
| `home` | 메인 배너, 인기 부스(Caffeine 캐시) | — (조회 전용) |
| `map` | 축제장 맵 위치(무대·부스 좌표) | `MapLocation` |
| `monitoring` | 헬스 체크, 최근 에러 로그, Grafana 알림 웹훅 | — (운영 전용) |

---

## API

- **Swagger UI**: 앱 기동 후 [http://localhost:8080/swagger-ui.html](http://localhost:8080/swagger-ui.html) (프로덕션에서는 비활성화)
- **OpenAPI 스펙**: `/v3/api-docs`
- 사용자용 엔드포인트는 `/api/**`, 운영진용은 `/api/admin/**`. 어드민 엔드포인트는 세션 인증과 역할 검증이 필요하다.

---

## 시작하기 (로컬 개발)

### 사전 요구사항

- **JDK 17** (Temurin 권장) — `java -version`으로 확인
- **Docker Desktop** — MySQL·Redis 컨테이너 기동용

### 1. MySQL · Redis 띄우기

```bash
docker compose up -d      # MySQL(:3307) + Redis(:6379) 백그라운드 기동
docker compose ps         # 상태 확인 (Up + healthy 면 OK)
docker compose down       # 종료 (데이터 보존)
docker compose down -v    # 데이터까지 완전 초기화
```

`docker-compose.yml` 기본 접속 정보:

| 항목 | MySQL | Redis |
| --- | --- | --- |
| Host | `localhost` | `localhost` |
| Port | `3307` | `6379` |
| Database | `daedongje` | — |
| Username / Password | `daedongje` / `daedongje` | — |

### 2. 애플리케이션 실행

`application.yaml`의 datasource·Redis 기본값이 위 docker-compose 컨테이너에 맞춰져 있어 **별도 환경변수 없이 바로 실행**된다.

```bash
./gradlew bootRun
```

운영(RDS) 등 다른 DB를 쓸 때만 환경변수로 오버라이드한다:

```bash
DB_URL=jdbc:mysql://my-rds-endpoint:3306/daedongje \
DB_USERNAME=produser \
DB_PASSWORD=secret \
./gradlew bootRun
```

### 3. 확인

[http://localhost:8080/swagger-ui.html](http://localhost:8080/swagger-ui.html) 에서 API 문서를 확인한다.

---

## 환경 변수 & 프로파일

환경 의존 값은 yaml에 직접 박지 않고 `${ENV:default}` 형태로 참조한다. 환경변수만 맞추면 하나의 jar가 모든 환경에서 동작한다. 운영 환경변수 예시는 [`.env.example`](.env.example) 참고 (DB, Redis, 세션 쿠키, AWS S3, 모니터링 웹훅, Grafana Cloud 등).

| 프로파일 | 활성화 | 용도 |
| --- | --- | --- |
| (default) | `./gradlew bootRun` | 로컬 개발 — docker-compose MySQL/Redis 기본값 사용 |
| `dev` | `--spring.profiles.active=dev` | 스테이징 (현재 빈 스텁) |
| `prod` | `SPRING_PROFILES_ACTIVE=prod` | 프로덕션 — Swagger 비활성화, 세션 쿠키 `Secure; SameSite=None`, ECS JSON 구조화 로그 |

적용 순서: `application.yaml`(공통) → `application-{프로파일}.yaml`(오버라이드).

---

## 데이터베이스 (Flyway)

스키마 변경은 코드와 함께 버전 관리된다. 마이그레이션 파일은 `src/main/resources/db/migration/`에 위치하며, 앱 기동 시 Flyway가 미적용 파일을 순서대로 실행한다.

### 새 마이그레이션 추가

1. 현재 디렉토리에서 가장 큰 `V` 번호 + 1 확인
2. `V{번호}__{스네이크_케이스_설명}.sql` 생성 (예: `V40__add_booth_email.sql`)
3. SQL 작성 후 앱 기동 → 자동 적용 (`flyway_schema_history` 테이블에서 확인)

> 엔티티에는 `BaseEntity`의 `created_at` / `updated_at DATETIME(6) NOT NULL` 컬럼을 마이그레이션 SQL에 반드시 함께 정의한다. `ddl-auto=validate`가 컬럼 부재를 즉시 잡아낸다.

### 절대 하지 말 것

- ❌ **이미 머지된 `V` 파일 수정** — 적용된 환경에서 재실행되지 않아 환경 간 불일치 발생. 변경은 새 `V` 파일로.
- ❌ **버전 번호 건너뛰기** — Flyway는 순차 적용한다.
- ❌ **로컬에서 테이블 직접 생성** — 팀원과 스키마 불일치 발생.

---

## 테스트

```bash
./gradlew test
```

- **H2 인메모리 DB**(`MODE=MySQL`)를 사용 — Docker MySQL 없이 즉시 실행 가능
- 테스트에서는 Flyway 비활성, JPA `ddl-auto=create-drop`으로 엔티티 정의에서 스키마 자동 생성
- (운영 마이그레이션 흐름 검증은 추후 Testcontainers 통합 테스트로 보강 예정)

---

## 배포 & 운영

### 브랜치 전략

| 브랜치 | 용도 |
| --- | --- |
| `main` | 프로덕션 (리드 승인 후 머지) |
| `dev` | 통합 브랜치 (주간 스테이징 배포) |
| `feature/*` | 기능 개발 (Linear 이슈 기반 자동 생성) |
| `hotfix/*` | 프로덕션 긴급 수정 |

### CI / CD (GitHub Actions)

- **CI** (`.github/workflows/ci.yml`) — `dev`·`main` 대상 PR/Push에서 Flyway 버전 중복 검사 → Gradle wrapper 검증 → 컴파일 → 단위 테스트(`./gradlew test`). 실패 시 테스트 리포트 업로드.
- **배포** (`.github/workflows/deploy.yml`) — `main` Push(또는 수동 dispatch) 시 Docker 이미지 빌드·푸시 → EC2 SSH 배포(`docker-compose.prod.yml`) → `/actuator/health` 폴링으로 검증, 실패 시 직전 이미지로 자동 롤백.

### 운영 엔드포인트 (Actuator)

| 엔드포인트 | 용도 |
| --- | --- |
| `/actuator/health` | 헬스 체크 (배포 검증·로드밸런서) |
| `/actuator/prometheus` | Prometheus 메트릭 (Grafana Alloy가 수집) |

프로덕션에서는 콘솔 로그를 ECS JSON으로 구조화해 Alloy가 Loki로 수집하고, HTTP latency 히스토그램으로 Grafana 대시보드의 p95/p99 패널을 그린다. Grafana 알림은 `/api/monitoring/webhooks/alerts`로 수신한다.

---

## 기여

브랜치·이슈·PR·커밋 규칙은 [`CONTRIBUTING.md`](CONTRIBUTING.md)를 따른다. 핵심만 요약하면:

- 브랜치는 **Linear 이슈에서 자동 생성된 이름**(`feature/BACK-xx-...`, 영어)을 그대로 사용하고 **`dev`에서 분기**한다.
- 커밋 메시지는 컨벤션 접두사(`feat:` `fix:` `docs:` `chore:` `refactor:` `test:`)를 사용한다.
- PR은 `.github/PULL_REQUEST_TEMPLATE.md`를 따르고, 각 팀 팀장 리뷰를 받는다.

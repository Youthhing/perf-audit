---
name: perf-audit
version: 0.3.0
description: |
  API performance auditor for Spring REST APIs.
  Pipeline: Stage A classify APIs (git diff + decision tree) → Stage B k6 load test (roadmap) → Stage C DB audit.
  Stage C detects N+1 and slow queries with EXPLAIN ANALYZE and suggests JPA fixes.
  Triggers: "API 분류", "성능 테스트 준비", "부하 테스트", "N+1", "slow query", "perf audit", "api benchmark".
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
  - AskUserQuestion
---

# /perf-audit — API Performance Auditor

Spring REST API의 성능을 커밋 전에 점검하기 위한 스킬.
전체 파이프라인은 **Stage A (분류) → Stage B (k6 부하 테스트) → Stage C (DB audit)**.
API 유형별로 부하 테스트의 목적과 프리셋이 달라야 하기 때문에, 파이프라인의 앞단에서
각 API를 분류하는 것이 핵심이다.

## Argument Routing

Parse the user's invocation:
- `/perf-audit` or `/perf-audit --classify` → **Stage A (분류)** 실행 — 아래 Stage A 섹션의 모든 단계
- `/perf-audit --k6` → "Stage B (k6 부하 테스트)는 로드맵 중입니다. 먼저 분류를 실행하세요: `/perf-audit`"
- `/perf-audit --db` or `/perf-audit --phase1` → **Stage C (DB Audit)** 실행 — 기존 DB audit 워크플로우 (Stage C 섹션 전체)
- `/perf-audit --all` → "분류 → k6 → DB audit 파이프라인 자동 실행은 로드맵 중입니다. 현재는 단계별로 실행해주세요."
- `/perf-audit --reset` → Reset performance_schema (Stage C의 Step R), then stop
- `/perf-audit --sql "<SQL>"` → Stage C의 Step 5로 바로 진입, Step 3-4 스킵

---

# Stage A: API Classification

목표: 변경된 API 후보를 자동 추출하고 사용자 검증을 거친 뒤, 각 API에 결정 트리를
적용해 7개 기본 패턴 × 2개 플래그로 분류한다. 이 분류 결과는 이후 Stage B(k6)가
패턴별 시나리오 프리셋을 선택하는 입력이 된다.

이번 버전에서는 **분류 단계의 산출물(Markdown 리포트)까지만** 생성한다.
k6 스크립트 자동 생성과 Stage C 연동은 로드맵.

## Step A-1: API Discovery from Git Diff

변경된 Controller/Service 파일에서 엔드포인트 후보를 추출한다.

```bash
# 1차: 브랜치 vs main 비교
CANDIDATES=$(git diff main...HEAD --name-only 2>/dev/null \
  | grep -E '(Controller|Service)\.(java|kt)$')

# 2차: staged + unstaged
if [ -z "$CANDIDATES" ]; then
  CANDIDATES=$( (git diff --name-only; git diff --cached --name-only) \
    | sort -u | grep -E '(Controller|Service)\.(java|kt)$')
fi

# 3차 (fallback): 최근 5개 커밋
if [ -z "$CANDIDATES" ]; then
  CANDIDATES=$(git log -5 --name-only --pretty=format: \
    | sort -u | grep -E '(Controller|Service)\.(java|kt)$')
fi

echo "후보 파일:"
echo "$CANDIDATES"
```

각 Controller 파일에서 HTTP 엔드포인트 추출 (Grep 사용):
- `@GetMapping` / `@PostMapping` / `@PutMapping` / `@PatchMapping` / `@DeleteMapping`
- `@RequestMapping(method = RequestMethod.XXX, ...)`
- 클래스 레벨 `@RequestMapping("/base/path")`과 메서드 레벨 path를 합쳐 **full path** 구성

각 후보를 다음 형태로 수집:
```
HTTP | fullPath | controllerFile:line | methodName
```

**후보 0개인 경우:**
```
ℹ️  변경된 Controller/Service가 없습니다.
    특정 API를 직접 입력하면 분류해드립니다. (예: GET /api/users/{id})
```
→ AskUserQuestion으로 수동 입력 요청 후 Step A-2로 이동. 입력도 없으면 종료.

## Step A-2: User Validation

추출한 후보 목록을 AskUserQuestion으로 검증받는다.

```
> 이 API들을 분류할까요? 체크 해제하면 제외, "Other"로 다른 API 추가 가능.
>
> [x] GET  /api/questions/{id}   (QuestionController.java:45 — getQuestion)
> [x] POST /api/questions         (QuestionController.java:60 — createQuestion)
> [x] PUT  /api/questions/{id}    (QuestionController.java:80 — updateQuestion)
```

- multiSelect=true로 기본 전체 선택 상태 제시
- 사용자가 "Other"로 입력한 API 경로는 Grep으로 실제 Controller 메서드를 찾아 합류
  - 찾지 못하면 "⚠️ {path}에 해당하는 Controller 메서드를 찾지 못했습니다. 직접 파일 경로를 지정해주세요."

사용자가 다른 API 분석을 원할 수 있음을 명시적으로 허용한다 —
git diff 기반 추출은 "출발점"일 뿐, 분류 대상 결정권은 사용자에게 있다.

## Step A-3: Per-API Source Trace

선택된 각 API에 대해 분류에 필요한 정보를 수집한다.

1. **Controller 메서드** 본문 읽기 (Read)
   - 메서드 시그니처: HTTP 어노테이션, `@PathVariable`, `@RequestParam`, `@RequestBody`, `@ModelAttribute`
   - 반환 타입: `Page<T>` / `Slice<T>` / `List<T>` / `Optional<T>` / 단일 DTO / `SseEmitter` / `Flux` / `ResponseBodyEmitter`
   - 서비스 호출 식별 (의존성 + 메서드명)

2. **Service 메서드** 본문 읽기
   - 클래스 레벨 + 메서드 레벨 `@Transactional(readOnly = ?)` 확인
   - `@Async`, `@Scheduled` 존재 여부
   - 호출하는 Repository 목록 (서로 다른 인스턴스 수)
   - 호출하는 다른 Service 컴포넌트
   - 외부 호출 흔적: `RestTemplate`, `WebClient`, `RestClient`, `@FeignClient`, AWS SDK
   - `saveAll` / `deleteAll` / `save` / `delete` / `@Modifying @Query` 호출

3. **Repository 인터페이스** 스캔 (Aggregation 판정용)
   - `@Query` 내 `GROUP BY`/`COUNT(`/`SUM(`/`AVG(`/`MIN(`/`MAX(`
   - 메서드명 패턴: `count*` / `sum*` / `get*Stats` / `get*Summary` / `aggregate*`
   - `Specification` / QueryDSL 사용 여부 (Search Read 판정용)

Grep 예시:
```bash
# Controller → Service 체인 추적
grep -rn "class {ControllerName}" --include="*.java" --include="*.kt"
grep -n "{serviceMethodName}(" <service_file>

# External call 감지
grep -rn "RestTemplate\|WebClient\|RestClient\|@FeignClient" <service_files>
```

## Step A-4: Classify (Decision Tree)

**원칙:** 아래 표들을 **위에서부터 순서대로** 적용한다. 먼저 매치되는 분류로 확정.
모든 API는 "7개 기본 패턴 중 하나 + 0~2개 플래그" 또는 "SKIP/UNSUPPORTED"로 단일 분류된다.

### 0. 사전 제외 필터 (SKIP/UNSUPPORTED)

| 조건 | 판정 | 사유 |
|---|---|---|
| `@Async` 어노테이션 존재 | SKIP | 응답시간 측정 무의미 |
| `@Scheduled` 어노테이션 존재 | SKIP | API 아님 |
| 반환 타입이 `SseEmitter` / `ResponseBodyEmitter` / `Flux`(SSE 용도) | UNSUPPORTED | 측정 패러다임 다름 |
| `WebSocketHandler` 구현체 | UNSUPPORTED | 측정 패러다임 다름 |
| 메서드 본문이 `kafkaTemplate.send()` / `rabbitTemplate.convertAndSend()` 호출만 | SKIP | 큐 발행 전용 |

해당되지 않으면 1단계로.

### 1. 1차 분류: 읽기 vs 쓰기

| 판정 | 조건 (둘 중 하나라도 해당) |
|---|---|
| **쓰기** | • HTTP가 `POST`/`PUT`/`PATCH`/`DELETE`<br>• `@Transactional`(readOnly=false 또는 속성 없음)<br>• Repository의 `save`/`delete`/`@Modifying @Query` 호출 |
| **읽기** | • HTTP가 `GET`<br>• `@Transactional(readOnly = true)`<br>• 상태 변경 쿼리 없음 |

**예외 규칙:**
- `POST`지만 상태 변경 쿼리 부재 + 반환값이 조회 결과 → **읽기로 재분류** (검색 조건 전달용 POST)
- `GET`이지만 카운터 증가 같은 부수 쓰기 있음 → **쓰기로 분류**

### 2. 읽기 세부 분류 (우선순위: Aggregation > Search > List > Single)

| 순위 | 패턴 | 판정 조건 | 예시 |
|---|---|---|---|
| 1 | **Aggregation** | `@Query`에 `GROUP BY`/`COUNT(`/`SUM(`/`AVG(`/`MIN(`/`MAX(` **또는** 메서드명 `count*`/`sum*`/`get*Stats`/`get*Summary`/`aggregate*` **또는** 반환 DTO 필드 과반이 숫자형 | `GET /orders/stats?from=...` |
| 2 | **Search Read** | `@RequestParam` 3개 이상 **또는** `@ModelAttribute` 검색 DTO **또는** Specification/QueryDSL **또는** 메서드명 `search*`/`find*By*And*By*` | `GET /products?category=...&price=...&sort=...` |
| 3 | **List Read** | 반환 타입 `Page<T>`/`Slice<T>`/`List<T>` **또는** `Pageable` 파라미터 | `GET /products`, `GET /users/{id}/orders` |
| 4 | **Single Read** | 위 3개 미해당 (기본값) — 반환 타입 단일 엔티티/`Optional<T>`/단일 DTO, `@PathVariable` 식별자 | `GET /users/{id}` |

### 3. 쓰기 세부 분류 (우선순위: Delete > Bulk > Single)

| 순위 | 패턴 | 판정 조건 | 예시 |
|---|---|---|---|
| 1 | **Delete** | HTTP가 `DELETE` **또는** 메서드명 `delete*`/`remove*` | `DELETE /users/{id}` |
| 2 | **Bulk Write** | `@RequestBody`가 `List<T>`/`Collection<T>`/배열 **또는** 메서드명 `bulk*`/`batch*`/`*All` **또는** `saveAll()`/`deleteAll()` | `POST /products/bulk` |
| 3 | **Single Write** | 위 2개 미해당 (기본값) — `@RequestBody` 단일 DTO, 단일 엔티티 대상 | `POST /users`, `PUT /products/{id}` |

### 4. 플래그 판정 (독립, 중복 가능)

#### 🔶 Composite 플래그

**부여 조건 (모두 만족):**
- 쓰기 계열 패턴 (Single Write / Bulk Write / Delete)
- 서비스 메서드에 `@Transactional` 존재
- 다음 중 하나:
  - 서로 다른 2개 이상 Repository 호출
  - 다른 Service 컴포넌트 호출 (Service 간 의존성)
  - 단일 메서드 내에서 2개 이상 테이블에 쓰기

**제외 (Composite 미부여):**
- cascade로만 자동 쓰기 발생
- 감사 로그(audit log) 같은 AOP 기반 부수 쓰기
- `findById` → `save` 1개 테이블 대상 "조회 후 수정"

#### 🔷 External Call 플래그

**부여 조건 (하나라도 해당):**
- `RestTemplate` / `WebClient` / `RestClient` 주입 또는 사용
- `@FeignClient` 인터페이스 주입
- AWS SDK 클라이언트 (`S3Client`, `SesClient` 등)
- 외부 SDK (결제사, SMS 등)

**제외:** S3 **presigned URL 발급만** 하는 경우 (실제 전송 없음).

### 5. 출력 형식

```
{패턴명} [+ 🔶 Composite] [+ 🔷 External Call]

예시:
- Single Read
- List Read
- Search Read
- Aggregation
- Single Write
- Single Write + 🔶 Composite
- Single Write + 🔶 Composite + 🔷 External Call
- Bulk Write + 🔶 Composite
- Delete + 🔶 Composite
- SKIP / UNSUPPORTED
```

### 6. 결정 트리 (의사코드)

```
classify(controllerMethod, serviceMethod):
  # 0. 제외 필터
  if hasAsync or hasScheduled: return SKIP
  if returnsStreaming or isWebSocket: return UNSUPPORTED
  if onlyPublishesToQueue: return SKIP

  # 1. 읽기/쓰기
  isWrite = writeHttpMethod
         or mutableTransactional
         or callsWriteRepository
  # 예외 적용
  if httpPOST and noStateMutation and returnsQueryResult: isWrite = False
  if httpGET and hasCounterIncrement:                     isWrite = True

  # 2-3. 세부
  if isWrite: pattern = Delete > BulkWrite > SingleWrite (우선순위 순)
  else:       pattern = Aggregation > Search > List > Single

  # 4. 플래그
  flags = []
  if isWrite and isComposite: flags.append("Composite")
  if hasExternalCall:         flags.append("ExternalCall")

  return pattern, flags
```

### 7. 분류 예시 (참고용)

| 코드 특징 | 분류 |
|---|---|
| `GET /users/{id}` → `userRepository.findById(id)` | Single Read |
| `GET /products?page=0&size=20` → `Page<Product>` | List Read |
| `GET /products?category=X&minPrice=Y&sort=Z` (파라미터 3+) | Search Read |
| `GET /orders/revenue?year=2024` → `SUM(amount)` | Aggregation |
| `POST /users` + `userRepository.save()` 단일 | Single Write |
| `POST /products/batch` + `saveAll(List<>)` | Bulk Write |
| `DELETE /users/{id}` | Delete |
| `POST /orders` (주문 + 재고차감 + 포인트 적립, 3개 테이블) | Single Write + 🔶 Composite |
| `POST /payments` (외부 PG + 결제 저장) | Single Write + 🔶 Composite + 🔷 External Call |
| `POST /notifications/send` (Feign으로 외부 알림) | Single Write + 🔷 External Call |
| `@Scheduled fun syncDaily()` | SKIP |
| `GET /events/stream` → `SseEmitter` | UNSUPPORTED |

각 API마다 판정 이유를 **한 줄로** 남긴다. 예:
> `Single Write + 🔶 Composite`: `@Transactional`, PaymentRepo/OrderRepo/PointRepo (3개) 호출

## Step A-5: Output Report

저장 경로: `~/.gstack/perf-audit-classify-{yyyymmdd-HHMMss}.md`

템플릿:

````markdown
# API Classification
**브랜치:** {current_branch}
**기준:** {discovery_source — e.g. "git diff main...HEAD"}
**분류 대상:** {N}개
**생성 시각:** {date}

## 요약

| HTTP | Path | 분류 | 플래그 | 근거 |
|------|------|------|--------|------|
| GET  | /api/questions/{id} | Single Read  | —                         | @PathVariable + Optional<Question> |
| POST | /api/questions      | Single Write | 🔶 Composite              | @Transactional, 3개 Repository |
| POST | /api/payments       | Single Write | 🔶 Composite + 🔷 External | WebClient PG + 결제 저장 |
| GET  | /api/stats          | Aggregation  | —                         | GROUP BY + COUNT |

## 상세 분류

### POST /api/payments — Single Write + 🔶 Composite + 🔷 External Call
- **파일:** `PaymentController.java:42` → `PaymentService.requestPayment:88`
- **판정 근거:**
  - 1차 — POST + `@Transactional` → 쓰기
  - 세부 — `@RequestBody PaymentRequest`(단일) → Single Write
  - 🔶 Composite — PaymentRepo.save + OrderRepo.findById + PointRepo.deduct (3개 Repo)
  - 🔷 External — WebClient로 PG사 결제 API 호출

### GET /api/stats — Aggregation
- **파일:** `StatsController.java:18` → `StatsService.getOrderRevenue:24`
- **판정 근거:**
  - 1차 — GET + `@Transactional(readOnly=true)` → 읽기
  - 세부 — `@Query`에 `SUM(amount) GROUP BY month` → Aggregation (우선순위 1)

## 다음 단계 (로드맵)

- [ ] **Stage B** — `/perf-audit --k6`: 패턴별 k6 시나리오 프리셋
  - Single Read: 고 RPS, 낮은 VU, 짧은 응답 p95 임계
  - Bulk Write: 낮은 RPS, 높은 VU, TX/락 경합 관측
  - Aggregation: p95 임계 완화, 쿼리 시간 관측 강화
  - +Composite: 더 낮은 RPS + 에러율 임계 강화
  - +External Call: 외부 의존성 mocking 전략 명시
- [ ] **Stage C** — `/perf-audit --db`: 부하 테스트 중 잡힌 slow query 분석 (기존 DB audit)
````

리포트 저장 후 터미널 출력:
```
📄 분류 리포트: ~/.gstack/perf-audit-classify-{timestamp}.md

✅ Stage A 완료.
  - 분류된 API: {N}개
  - SKIP/UNSUPPORTED: {M}개

다음 단계:
  Stage B (k6 시나리오) — 로드맵 진행 중
  Stage C (DB audit) — 지금 실행: /perf-audit --db
```

---

# Stage C: DB Audit

DB-first 감사 워크플로우. `--db` 또는 `--phase1` 인자로 실행된다.
performance_schema에서 슬로우 쿼리를 추출하고 EXPLAIN ANALYZE를 돌려
N+1/Full Scan 의심 쿼리에 JPA 수정 코드를 제안한다.

## Step 0: Load Config

Search for `.gstack-perf.yaml` starting from the current directory upward:

```bash
_CFG_FILE=""
_DIR=$(pwd)
while [ "$_DIR" != "/" ]; do
  if [ -f "$_DIR/.gstack-perf.yaml" ]; then
    _CFG_FILE="$_DIR/.gstack-perf.yaml"
    echo "CONFIG: $_CFG_FILE"
    break
  fi
  _DIR=$(dirname "$_DIR")
done
[ -z "$_CFG_FILE" ] && echo "CONFIG: NOT_FOUND"
```

**If NOT_FOUND:** Run the Config Setup wizard below, then proceed.

**If found:** Read and parse the YAML. Extract:
- `db_name` (required)
- `db_user` (default: root)
- `db_password` (default: "" — or env `$PERF_DB_PASSWORD`)
- `docker_container` (optional — if set, use `docker exec {container} mysql`)
- `db_host` (default: 127.0.0.1, ignored if docker_container is set)
- `db_port` (default: 3306, ignored if docker_container is set)
- `slow_query_ms` (default: 500)
- `top_n_queries` (default: 10)
- `framework` (default: spring)

**Build the MySQL command — define a shell function:**

```bash
# Build mysql function based on config
if [ -n "{docker_container}" ]; then
  mysql_cmd() { docker exec {docker_container} mysql -u{db_user} -p{db_password} "$@" 2>&1; }
  mysql_db()  { docker exec {docker_container} mysql -u{db_user} -p{db_password} {db_name} "$@" 2>&1; }
else
  MYSQL_BIN=$(which mysql 2>/dev/null)
  [ -z "$MYSQL_BIN" ] && MYSQL_BIN=$(ls /opt/homebrew/opt/mysql*/bin/mysql /usr/local/bin/mysql /usr/bin/mysql 2>/dev/null | head -1)
  if [ -z "$MYSQL_BIN" ]; then
    echo "❌ mysql client not found. Install with: brew install mysql-client"
    exit 1
  fi
  mysql_cmd() { "$MYSQL_BIN" -h {db_host} -P {db_port} -u {db_user} -p{db_password} "$@" 2>&1; }
  mysql_db()  { "$MYSQL_BIN" -h {db_host} -P {db_port} -u {db_user} -p{db_password} {db_name} "$@" 2>&1; }
fi

# Test the function works
mysql_cmd -e "SELECT 1;" > /dev/null 2>&1 && echo "✓ MySQL 연결 확인" || echo "❌ MySQL 연결 실패"
```

In all subsequent steps, use `mysql_cmd` (no DB selected) and `mysql_db` (with DB selected) instead of `$MYSQL` and `$MYSQL_DB`.

---

## Config Setup Wizard (only when .gstack-perf.yaml is missing)

Use AskUserQuestion to collect:

> **perf-audit 처음 실행이에요.** 프로젝트 DB 설정을 입력해주세요.
>
> 이 설정은 `.gstack-perf.yaml`에 저장되고 `.gitignore`에 자동 추가됩니다.

Ask for:
1. DB 이름 (예: myapp_dev)
2. DB 접속 방법: A) Docker 컨테이너 B) 직접 TCP 연결
3. If A: 컨테이너 이름 (예: mysql8) + DB user/password
4. If B: host/port + DB user/password

Then write to `.gstack-perf.yaml` in the project root:

```yaml
# .gstack-perf.yaml — perf-audit configuration
# WARNING: Contains DB credentials. Do NOT commit this file.

db_name: "{db_name}"
db_user: "{db_user}"
db_password: "{db_password}"    # or use env var: PERF_DB_PASSWORD

# Docker MySQL (set container name if MySQL runs in Docker, leave empty otherwise)
docker_container: "{container_or_empty}"

# Direct TCP (only used when docker_container is empty)
db_host: "127.0.0.1"
db_port: 3306

# Audit settings
slow_query_ms: 500     # Queries slower than this (ms) are flagged
top_n_queries: 10      # Analyze top N slowest queries

# Framework (affects fix code generation)
framework: "spring"    # spring | django | rails | fastapi
```

Auto-add to `.gitignore`:
```bash
grep -qF ".gstack-perf.yaml" .gitignore 2>/dev/null || echo ".gstack-perf.yaml" >> .gitignore
echo "✓ .gstack-perf.yaml added to .gitignore"
```

---

## Step 1: Session Brief

Use AskUserQuestion (single call, two questions):

> **무엇을 분석할까요?**
>
> Q1. 방금 구현한 기능이나 성능이 걱정되는 API가 있나요?  
> (예: "유저 피드 조회", "문제 세트 로딩", "검색 API")
>
> Q2. 분석 목적:
> - A) N+1 같은 특정 문제 찾기
> - B) PR 전 전체 성능 기준선 확인
> - C) 특정 API가 느린 이유 파악

Store as `BRIEF_FEATURE`, `BRIEF_PURPOSE`.

Extract domain keywords from Q1 answer for Phase 0:
- "문제 세트 로딩" → `BRIEF_DOMAIN="question_set,question"`
- "유저 피드 조회" → `BRIEF_DOMAIN="user,feed,post"`
- 모르겠으면 모든 Entity 스캔

---

## Step 1.5: Phase 0 — Test Data Setup

> 이 단계는 performance_schema에 의미있는 쿼리 데이터가 쌓이도록
> DB에 더미 데이터를 삽입합니다. 감사 완료 후 자동으로 삭제됩니다.

**1a. Orphan cleanup 파일 확인:**

```bash
CLEANUP_FILE="$HOME/.gstack/perf-audit-cleanup-${SLUG}-${DB_NAME}.sql"
```

파일이 존재하면:
```
⚠️  이전 실행에서 생성된 더미데이터가 남아 있습니다.
    파일: $CLEANUP_FILE
    먼저 정리할까요?
```
→ 사용자 동의 시: `mysql_db < "$CLEANUP_FILE" 2>&1` 실행 후 파일 삭제
→ 거부 시: 경고 표시 후 계속

**1b. DB 권한 확인:**

```bash
mysql_db -e "SHOW GRANTS FOR CURRENT_USER();" 2>&1
```

INSERT/DELETE 권한 없으면:
```
⚠️  INSERT 권한이 없어 더미데이터를 생성할 수 없습니다.
    read-only 모드로 진행합니다 (기존 데이터로만 분석).

    권한 부여 방법:
    GRANT INSERT, DELETE ON {db_name}.* TO '{db_user}'@'%';
```
→ read-only 모드로 Step 3으로 진행 (더미 데이터 없이)

**1c. 소스코드 분석 — 비즈니스 맥락 파악:**

`BRIEF_DOMAIN` 키워드를 기반으로 프로젝트 소스에서 관련 Entity 찾기:

```bash
# Entity 파일 탐색
find . -name "*Entity*.java" -o -name "*entity*.java" | head -20
# BRIEF_DOMAIN 키워드와 매칭되는 Entity 우선 선택
grep -rl "BRIEF_DOMAIN_KEYWORD" src/ --include="*.java" | head -10
```

각 Entity 파일을 읽어서 파악:
- 테이블명 (`@Table(name = "...")`)
- 컬럼 및 타입 (`@Column`)
- FK 관계 (`@ManyToOne`, `@OneToMany`, `@JoinColumn`)
- NOT NULL 제약 (`nullable = false`)
- Enum 타입 (`@Enumerated`)
- 기본값 (`@Builder.Default`, `@Column(columnDefinition = "...")`)

**1d. INSERT SQL 생성:**

파악한 Entity 구조를 바탕으로 의미있는 더미데이터 INSERT SQL 생성.

규칙:
- NOT NULL 컬럼은 반드시 값 포함
- Enum은 해당 Enum 클래스를 읽어서 유효한 값 사용
- FK는 부모 레코드를 먼저 삽입
- N+1이 감지되려면 **최소 20개 이상** 레코드 필요 (예: 부모 5개, 자식 4개씩)
- 더미 식별을 위해 문자열 필드에 `[perf-audit-dummy]` prefix 포함
- AUTO_INCREMENT PK는 INSERT 후 `LAST_INSERT_ID()`로 ID 수집

**1e. Cleanup 파일에 즉시 기록 (INSERT 전):**

각 INSERT 직전에 cleanup SQL을 파일에 추가:

```bash
# INSERT 전에 DELETE 구문 먼저 저장
echo "DELETE FROM {table} WHERE id = LAST_INSERT_ID();" >> "$CLEANUP_FILE"
# 또는 식별 가능한 경우:
echo "DELETE FROM {table} WHERE {string_col} LIKE '[perf-audit-dummy]%';" >> "$CLEANUP_FILE"
```

**1f. INSERT 실행:**

```bash
mysql_db -e "{generated_insert_sql}" 2>&1
```

FK 제약 오류 시:
```
❌ INSERT 실패: {error}
   테이블: {table}
   원인: FK 제약 조건 위반 또는 NOT NULL 누락
   수동 INSERT SQL:
   {insert_sql}
```
→ 오류 난 테이블 스킵하고 다음 테이블로 계속

**1g. 삽입 완료 후 확인:**

```bash
mysql_db -e "SELECT COUNT(*) as inserted FROM {main_table} WHERE {string_col} LIKE '[perf-audit-dummy]%';" 2>&1
```

```
✓ 더미데이터 생성 완료
  - {table_a}: {N}개 삽입
  - {table_b}: {N}개 삽입
  cleanup 파일: $CLEANUP_FILE
```

**1h. performance_schema 초기화 (더미 데이터 삽입 전 통계 초기화):**

```bash
mysql_cmd -e "TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;" 2>&1
echo "✓ performance_schema 초기화 — 이제부터 실행되는 쿼리만 측정됩니다."
```

**1i. 더미 데이터로 쿼리 발생시키기:**

삽입된 더미 데이터를 대상으로 분석 대상 테이블의 주요 SELECT 패턴 실행:

```bash
# 단건 조회 패턴 (N+1 감지용) — 각 ID마다 개별 조회
for id in $(mysql_db -e "SELECT id FROM {table} WHERE {col} LIKE '[perf-audit-dummy]%' LIMIT 10;" | tail -n +2); do
  mysql_db -e "SELECT * FROM {related_table} WHERE {fk_col} = $id;" > /dev/null
done

# 전체 목록 조회 패턴
mysql_db -e "SELECT * FROM {main_table};" > /dev/null
```

이 단계가 끝나면 performance_schema에 실제 쿼리 패턴이 쌓임 → Step 3으로 진행.

Store as `BRIEF_FEATURE` and `BRIEF_PURPOSE`.

---

## Step 2: Preflight Check

Run each check. Stop with a clear, actionable error if any fail.

**Check 1 — MySQL connection:**
```bash
mysql_cmd -e "SELECT VERSION();" 2>&1
```
- Connection refused → "❌ DB 연결 실패. docker_container 설정이 맞는지, 또는 host/port를 확인하세요."
- 5.x version → "❌ MySQL 8.0+ required for EXPLAIN ANALYZE. Current: {version}. Phase 1을 사용하려면 MySQL 8.0+이 필요합니다."
- Success → print "✓ MySQL {version} 연결 성공"

**Check 2 — performance_schema:**
```bash
mysql_cmd -e "SELECT VARIABLE_VALUE FROM performance_schema.global_variables WHERE VARIABLE_NAME='performance_schema';" 2>&1
```
- OFF → "❌ performance_schema 비활성화. MySQL 설정에서 `performance_schema=ON`으로 변경 후 재시작하세요."
- Success → print "✓ performance_schema ON"

**Check 3 — DB exists:**
```bash
mysql_cmd -e "USE {db_name};" 2>&1
```
- Error → "❌ DB '{db_name}'를 찾을 수 없습니다. db_name 설정을 확인하세요."

Print: "✓ 프리플라이트 완료. 분석 시작합니다."

---

## Step 3: Extract Slow Queries

```bash
mysql_db -e "
SELECT
  DIGEST_TEXT,
  COUNT_STAR                              AS exec_count,
  ROUND(AVG_TIMER_WAIT / 1000000000, 2)  AS avg_ms,
  ROUND(MAX_TIMER_WAIT / 1000000000, 2)  AS max_ms,
  ROUND(SUM_TIMER_WAIT / 1000000000, 2)  AS total_ms,
  SUM_ROWS_EXAMINED                       AS rows_examined,
  SUM_ROWS_SENT                           AS rows_sent
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = '{db_name}'
  AND DIGEST_TEXT NOT LIKE 'SELECT @@%'
  AND DIGEST_TEXT NOT LIKE 'SET %'
  AND DIGEST_TEXT NOT LIKE 'SHOW %'
  AND DIGEST_TEXT NOT LIKE 'TRUNCATE%'
  AND DIGEST_TEXT NOT LIKE 'ALTER%'
  AND AVG_TIMER_WAIT / 1000000000 >= {slow_query_ms}
ORDER BY SUM_TIMER_WAIT DESC
LIMIT {top_n_queries};
" 2>&1
```

**If no results:** Lower threshold to 50ms and retry once. If still empty:

```
ℹ️  슬로우 쿼리 데이터가 없습니다 (임계값: {slow_query_ms}ms).

아직 API를 호출하지 않았거나 모든 쿼리가 빠릅니다.

방법:
  1. 서버를 기동하고 API를 몇 번 호출한 뒤 다시 실행하세요.
  2. performance_schema를 초기화하고 싶다면: /perf-audit --reset
  3. 특정 SQL을 직접 분석하려면: /perf-audit --sql "SELECT ..."
```
Then stop.

---

## Step 4: N+1 Pattern Detection

Analyze the extracted queries. Flag as **⚠️ N+1 의심** when ALL of:
1. `exec_count >= 5`
2. `DIGEST_TEXT` contains `WHERE {column} = ?` with a single equality condition (single-row fetch pattern)
3. `rows_examined / rows_sent` is close to 1 (fetching one row at a time)

Also flag as **🔴 Full Scan 의심** when:
- `rows_examined` >> `rows_sent` (ratio > 10) AND `avg_ms` is high
- This indicates MySQL is reading many rows to find few results (missing index)

Print a quick summary before EXPLAIN:
```
📊 쿼리 분석 결과:
  총 {N}개 쿼리 | ⚠️ N+1 의심 {A}개 | 🔴 Full Scan 의심 {B}개 | ✅ 정상 {C}개
```

---

## Step 5: EXPLAIN ANALYZE (top 5 queries max)

For each query (limit 5 to avoid timeouts):

**5a. Reconstruct concrete SQL from DIGEST_TEXT:**

Replace `?` with real sample values from the DB:
```bash
# Extract table name from DIGEST_TEXT
# For WHERE {col} = ? pattern: get a sample value
mysql_db -e "SELECT {col} FROM {table} WHERE {col} IS NOT NULL LIMIT 1;" 2>&1
```

**5b. Check table schema:**
```bash
mysql_db -e "SHOW CREATE TABLE {table_name};" 2>&1
mysql_db -e "SHOW INDEX FROM {table_name};" 2>&1
```

**5c. Run EXPLAIN ANALYZE with timeout guard:**
```bash
mysql_db -e "
SET SESSION max_execution_time=10000;
EXPLAIN ANALYZE {reconstructed_sql}\G
" 2>&1
```

`max_execution_time=10000` = 10초 (밀리초). 초과 시 MySQL이 에러 반환:
- "max_execution_time exceeded" 메시지 → "⏱️ EXPLAIN ANALYZE 타임아웃. 쿼리가 복잡합니다. 스킵하고 다음 쿼리로."

**5d. Parse EXPLAIN ANALYZE output:**

Key patterns to identify:
| EXPLAIN output | Problem | Severity |
|----------------|---------|----------|
| `Full table scan` | Full scan, no index | 🔴 HIGH |
| `Nested loop` + high `actual rows` | N+1 join | ⚠️ MEDIUM |
| `rows=` very large, `actual rows=` small | Index miss | ⚠️ MEDIUM |
| `Filter:` present | Post-read filtering | ℹ️ LOW |
| `Index lookup` | Using index | ✅ OK |
| `Index scan` | Using index (range) | ✅ OK |

---

## Step 6: AI Fix Suggestions

For each problematic query, generate a fix block. Read the relevant source files to understand the actual entity/repository code:

```bash
# Find the entity and repository for the table
find . -name "*.java" | xargs grep -l "{table_name}" 2>/dev/null | head -5
```

Read those files, then generate fixes appropriate to the detected problem:

**Problem: N+1 (repeated single-row SELECT)**

```markdown
### ⚠️ N+1 감지: `{table}.{col} = ?` ({exec_count}회 실행)

**문제:** `{EntityName}`을 목록으로 로딩할 때 연관 엔티티를 건당 개별 쿼리로 조회합니다.
JPA LAZY 로딩의 기본 동작으로, 10개를 로딩하면 11번 쿼리가 실행됩니다.

**수정 방법 A — @EntityGraph (권장, 코드 변경 최소)**
```java
// {RepositoryName}.java
@EntityGraph(attributePaths = {"{관계필드명}"})
List<{EntityName}> findAll();

// 또는 특정 메서드에만 적용:
@EntityGraph(attributePaths = {"{관계필드명}"})
List<{EntityName}> findBy{조건}({타입} {파라미터});
```

**수정 방법 B — JPQL JOIN FETCH**
```java
// {RepositoryName}.java
@Query("SELECT e FROM {EntityName} e JOIN FETCH e.{관계필드명} WHERE ...")
List<{EntityName}> findAllWithRelation();
```

**수정 방법 C — @BatchSize (OneToMany에 효과적)**
```java
// {EntityName}.java
@OneToMany(mappedBy = "...", fetch = FetchType.LAZY)
@BatchSize(size = 100)
private List<{RelatedEntity}> items;
```

예상 개선: N+1 → 1~2 쿼리 (쿼리 수 {exec_count}배 감소)
```

**Problem: Full Table Scan**

```markdown
### 🔴 Full Scan: `{table}` ({rows_examined}행 읽어서 {rows_sent}행 반환)

**문제:** `{column}` 컬럼에 인덱스가 없어 테이블 전체를 스캔합니다.
데이터가 많아질수록 선형으로 느려집니다.

**수정 방법 A — 인덱스 추가 (DB)**
```sql
CREATE INDEX idx_{table}_{column} ON {table}({column});
```

**수정 방법 B — Entity 어노테이션**
```java
// {EntityName}.java
@Table(name = "{table}", indexes = {
    @Index(name = "idx_{table}_{column}", columnList = "{column}")
})
public class {EntityName} { ... }
```

JPA DDL 자동 생성 사용 중이면 어플리케이션 재시작 시 자동 적용됩니다.

예상 개선: O(n) → O(log n) 스캔 ({rows_examined}행 → 예상 {rows_sent}행)
```

---

## Step 7: Generate Report

Write to: `~/.gstack/perf-audit-{db_name}-{yyyyMMdd-HHmmss}.md`

```markdown
# API Performance Audit
**기능:** {BRIEF_FEATURE}  
**목적:** {BRIEF_PURPOSE}  
**날짜:** {date}  
**DB:** {db_name} (MySQL {version})  
**분석 쿼리:** {N}개

---

## 요약

| 순위 | 쿼리 (축약) | 실행 | 평균(ms) | 최대(ms) | 문제 |
|------|------------|------|---------|---------|------|
{table rows from analysis}

---

## 상세 분석

{fix blocks from Step 6, one per problematic query}

---

## 수정 우선순위

1. **즉시** (N+1): {list}
2. **단기** (인덱스): {list}
3. **모니터링** (정상이지만 주시 필요): {list}

---

## 다음 단계

- [ ] 수정 사항 적용 후 `/perf-audit --reset` 실행 (카운터 초기화)
- [ ] API 재호출 후 `/perf-audit` 재실행 (개선 확인)
- [ ] 리포트를 PR 설명에 첨부 (before/after 비교)
```

Print to terminal too. End with:
```
📄 리포트 저장: ~/.gstack/perf-audit-{db_name}-{timestamp}.md

✅ Phase 1 완료.
  - 분석된 쿼리: {N}개
  - 발견된 문제: {A}개 (N+1: {x}개, Full Scan: {y}개)
  - 수정 제안: 위 리포트 참조

다음:
  수정 적용 → /perf-audit --reset → API 재호출 → /perf-audit (개선 확인)
```

---

## Step 7.5: Cleanup Dummy Data

리포트 출력 직후, Phase 0에서 더미데이터를 삽입했다면 cleanup 실행.

```bash
CLEANUP_FILE="$HOME/.gstack/perf-audit-cleanup-${SLUG}-${DB_NAME}.sql"
```

파일이 존재하면:

```bash
echo "🧹 더미데이터 정리 중..."
mysql_db < "$CLEANUP_FILE" 2>&1
```

성공 시:
```
✓ 더미데이터 정리 완료.
```
→ `$CLEANUP_FILE` 삭제

실패 시:
```
⚠️  자동 cleanup 실패. 수동으로 실행하세요:
    mysql -u {db_user} -p {db_name} < $CLEANUP_FILE
```
→ 파일은 보존 (다음 실행 시 orphan 감지로 다시 안내)

Phase 0를 건너뛴 경우 (read-only 모드, 더미데이터 없음) → 이 단계 스킵.

---

## Step R: Reset (--reset argument)

```bash
mysql_cmd -e "TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;" 2>&1
echo "✓ performance_schema 초기화 완료."
echo "이제 서버를 실행하고 API를 호출한 뒤 /perf-audit을 다시 실행하세요."
```

---

## Notes for Implementers

- **Docker MySQL:** `docker_container` 설정 시 `docker exec {container} mysql ...` 형태로 실행. 포트 바인딩 불필요.
- **권한:** `performance_schema` 읽기에는 `PROCESS` 권한이 필요할 수 있음. 권한 오류 시: `GRANT PROCESS ON *.* TO '{user}'@'%';`
- **데이터 없음:** performance_schema는 실제 쿼리가 실행된 후에만 데이터를 가짐. 서버를 기동하고 API를 호출해야 함.
- **RDS/Aurora:** managed DB에서는 `performance_schema` 접근이 제한될 수 있음. `parameter group`에서 `performance_schema=1` 설정 필요.

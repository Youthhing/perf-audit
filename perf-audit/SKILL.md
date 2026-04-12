---
name: perf-audit
version: 0.2.0
description: |
  API performance auditor for MySQL + Spring REST APIs. Detects N+1 queries,
  slow queries, and missing indexes using EXPLAIN ANALYZE — before you commit.
  Phase 1: DB Audit (no server needed). Phase 2: k6 HTTP benchmark + worktree A/B (roadmap).
  Triggers: "성능 확인", "N+1", "쿼리 느려", "perf audit", "slow query", "api benchmark".
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
  - AskUserQuestion
---

# /perf-audit — API Performance Auditor

커밋 전에 실행해서 N+1, 풀 테이블 스캔, 슬로우 쿼리를 잡고 JPA/Spring 수정 제안까지 받는 스킬.

## Argument Routing

Parse the user's invocation:
- `/perf-audit` or `/perf-audit --phase1` → Phase 1 (DB Audit) — run all steps below
- `/perf-audit --phase2` → print "Phase 2 (k6 + worktree A/B)는 로드맵 중입니다. Phase 1을 먼저 실행하세요: `/perf-audit`"
- `/perf-audit --reset` → Reset performance_schema (Step R below), then stop
- `/perf-audit --sql "<SQL>"` → Skip to Step 5 with the provided SQL, skip Steps 3-4

---

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

**Build the MySQL command:**

```bash
# Check if docker_container is configured
if [ -n "{docker_container}" ]; then
  MYSQL="docker exec {docker_container} mysql -u{db_user} -p{db_password}"
  MYSQL_DB="docker exec {docker_container} mysql -u{db_user} -p{db_password} {db_name}"
else
  # Find mysql client
  MYSQL_BIN=$(which mysql 2>/dev/null)
  [ -z "$MYSQL_BIN" ] && MYSQL_BIN=$(ls /opt/homebrew/opt/mysql*/bin/mysql /usr/local/bin/mysql /usr/bin/mysql 2>/dev/null | head -1)
  [ -z "$MYSQL_BIN" ] && echo "❌ mysql client not found. Install with: brew install mysql-client" && exit 1
  MYSQL="$MYSQL_BIN -h {db_host} -P {db_port} -u {db_user} -p{db_password}"
  MYSQL_DB="$MYSQL_BIN -h {db_host} -P {db_port} -u {db_user} -p{db_password} {db_name}"
fi
```

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

Store as `BRIEF_FEATURE` and `BRIEF_PURPOSE`.

---

## Step 2: Preflight Check

Run each check. Stop with a clear, actionable error if any fail.

**Check 1 — MySQL connection:**
```bash
$MYSQL -e "SELECT VERSION();" 2>&1
```
- Connection refused → "❌ DB 연결 실패. docker_container 설정이 맞는지, 또는 host/port를 확인하세요."
- 5.x version → "❌ MySQL 8.0+ required for EXPLAIN ANALYZE. Current: {version}. Phase 1을 사용하려면 MySQL 8.0+이 필요합니다."
- Success → print "✓ MySQL {version} 연결 성공"

**Check 2 — performance_schema:**
```bash
$MYSQL -e "SELECT VARIABLE_VALUE FROM performance_schema.global_variables WHERE VARIABLE_NAME='performance_schema';" 2>&1
```
- OFF → "❌ performance_schema 비활성화. MySQL 설정에서 `performance_schema=ON`으로 변경 후 재시작하세요."
- Success → print "✓ performance_schema ON"

**Check 3 — DB exists:**
```bash
$MYSQL -e "USE {db_name};" 2>&1
```
- Error → "❌ DB '{db_name}'를 찾을 수 없습니다. db_name 설정을 확인하세요."

Print: "✓ 프리플라이트 완료. 분석 시작합니다."

---

## Step 3: Extract Slow Queries

```bash
$MYSQL_DB -e "
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
$MYSQL_DB -e "SELECT {col} FROM {table} WHERE {col} IS NOT NULL LIMIT 1;" 2>&1
```

**5b. Check table schema:**
```bash
$MYSQL_DB -e "SHOW CREATE TABLE {table_name};" 2>&1
$MYSQL_DB -e "SHOW INDEX FROM {table_name};" 2>&1
```

**5c. Run EXPLAIN ANALYZE with timeout guard:**
```bash
$MYSQL_DB --max_statement_time=10000 -e "
EXPLAIN ANALYZE
{reconstructed_sql};
" 2>&1
```

If timeout (10s): "⏱️ EXPLAIN ANALYZE 타임아웃 — 쿼리가 복잡합니다. 스킵하고 다음 쿼리로."

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

## Step R: Reset (--reset argument)

```bash
$MYSQL -e "TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;" 2>&1
echo "✓ performance_schema 초기화 완료."
echo "이제 서버를 실행하고 API를 호출한 뒤 /perf-audit을 다시 실행하세요."
```

---

## Notes for Implementers

- **Docker MySQL:** `docker_container` 설정 시 `docker exec {container} mysql ...` 형태로 실행. 포트 바인딩 불필요.
- **권한:** `performance_schema` 읽기에는 `PROCESS` 권한이 필요할 수 있음. 권한 오류 시: `GRANT PROCESS ON *.* TO '{user}'@'%';`
- **데이터 없음:** performance_schema는 실제 쿼리가 실행된 후에만 데이터를 가짐. 서버를 기동하고 API를 호출해야 함.
- **RDS/Aurora:** managed DB에서는 `performance_schema` 접근이 제한될 수 있음. `parameter group`에서 `performance_schema=1` 설정 필요.

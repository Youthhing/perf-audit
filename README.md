# perf-audit

**커밋 전에 실행하는 MySQL API 성능 감사 도구.**  
N+1 쿼리, 풀 테이블 스캔, 슬로우 쿼리를 자동으로 감지하고 JPA/Spring 수정 코드를 제안합니다.

서버 없이 동작합니다. DB 접속 정보만 있으면 됩니다.

---

## 왜 만들었나

API를 구현하고 나면 성능 문제가 있는지 커밋 전에 확인하고 싶은데:

- **Datadog, New Relic** — 배포 후에야 보임. 개발 루프 밖.
- **bullet_gem** — Rails 전용. Spring에는 없음.
- **JPA 로깅** — 쿼리는 보이지만 N+1인지 아닌지, 어떻게 고치는지 알려주지 않음.

perf-audit은 Claude Code 스킬로 동작해서, DB에 직접 붙어 `EXPLAIN ANALYZE`를 돌리고 어떤 쿼리가 문제인지, 어떤 JPA 코드로 고치는지까지 알려줍니다.

---

## 동작 방식

```
/perf-audit 실행
     │
     ├─ MySQL performance_schema에서 슬로우 쿼리 추출
     ├─ N+1 패턴 자동 감지 (동일 테이블 반복 단건 조회)
     ├─ EXPLAIN ANALYZE로 각 쿼리 분석
     ├─ 소스코드 읽어서 JPA 수정 코드 생성
     └─ Markdown 리포트 저장
```

**Phase 1 (현재):** DB Audit — 서버 불필요, MySQL 직접 분석  
**Phase 2 (로드맵):** k6 HTTP 벤치마크 + git worktree A/B 비교

---

## 요구사항

- [Claude Code](https://claude.ai/code) (claude.ai/code)
- MySQL 8.0+ (Docker 또는 직접 설치)
- Spring Boot + JPA 프로젝트 (다른 프레임워크도 동작하지만 fix 제안은 Spring 기준)

---

## 설치

### 1. 스킬 파일 복사

```bash
mkdir -p ~/.claude/skills/perf-audit
curl -o ~/.claude/skills/perf-audit/SKILL.md \
  https://raw.githubusercontent.com/YOUR_USERNAME/perf-audit/main/perf-audit/SKILL.md
```

또는 클론 후 복사:

```bash
git clone https://github.com/YOUR_USERNAME/perf-audit.git
mkdir -p ~/.claude/skills/perf-audit
cp perf-audit/perf-audit/SKILL.md ~/.claude/skills/perf-audit/SKILL.md
```

### 2. Claude Code에 스킬 등록

`~/.claude/CLAUDE.md`에 추가:

```markdown
## Skills

Use the `/perf-audit` skill for API performance auditing (N+1 detection, slow queries).
Available: `/perf-audit`, `/perf-audit --phase1`, `/perf-audit --reset`, `/perf-audit --sql "..."`
```

### 3. 프로젝트 설정

Spring 프로젝트 루트에 `.gstack-perf.yaml` 생성:

```bash
cp perf-audit/.gstack-perf.yaml.example /your/project/.gstack-perf.yaml
# 편집해서 DB 정보 입력
```

또는 `/perf-audit` 처음 실행 시 자동으로 생성해줍니다.

`.gitignore`에 추가 (자동으로 추가되지만 확인):
```
.gstack-perf.yaml
```

---

## 사용법

Spring 프로젝트 디렉토리에서 Claude Code를 열고:

```
/perf-audit
```

첫 실행 시 DB 설정을 물어봅니다. 이후에는 바로 분석 시작.

### 커맨드

| 커맨드 | 설명 |
|--------|------|
| `/perf-audit` | Phase 1 실행 (DB Audit) |
| `/perf-audit --phase1` | 위와 동일 |
| `/perf-audit --reset` | performance_schema 초기화 (API 재호출 전 사용) |
| `/perf-audit --sql "SELECT ..."` | 특정 SQL 직접 분석 |

### 기본 워크플로우

```bash
# 1. API 구현 완료
# 2. 서버 기동하고 해당 API 몇 번 호출 (또는 테스트 실행)
# 3. Claude Code에서:
/perf-audit

# 4. 리포트 확인 + 수정 적용
# 5. 다시 측정:
/perf-audit --reset
# → API 재호출
/perf-audit
```

---

## 실제 분석 예시

`QuestionSet` 목록 조회 API에서 발견된 N+1:

```
📊 쿼리 분석 결과:
  총 3개 쿼리 | ⚠️ N+1 의심 1개 | 🔴 Full Scan 의심 1개 | ✅ 정상 1개
```

```
### ⚠️ N+1 감지: `questions.question_set_id = ?` (6회 실행)

문제: QuestionSetEntity 목록을 로딩할 때 각 세트의 questions를
      건당 개별 쿼리로 조회합니다. 세트가 N개면 N+1번 쿼리가 실행됩니다.

수정 방법 A — @EntityGraph (권장)
```java
// QuestionEntityRepository.java
@EntityGraph(attributePaths = {"questionSet"})
List<QuestionEntity> findByQuestionSetId(Long questionSetId);
```

수정 방법 B — JPQL JOIN FETCH
```java
@Query("SELECT q FROM QuestionEntity q JOIN FETCH q.questionSet WHERE q.questionSet.id = :id")
List<QuestionEntity> findByQuestionSetIdWithJoin(@Param("id") Long id);
```

예상 개선: 6 쿼리 → 1 쿼리 (6배 감소)
```

---

## 설정 파일 전체 옵션

```yaml
# .gstack-perf.yaml

db_name: "myapp_dev"           # (필수) 분석할 데이터베이스 이름
db_user: "root"
db_password: ""                # 또는 환경변수 PERF_DB_PASSWORD

# Docker로 MySQL 실행 중이면:
docker_container: "mysql8"     # docker exec {container} mysql ... 형태로 실행
                               # 비어있으면 직접 TCP 연결

# 직접 TCP 연결 (docker_container 비어있을 때만 사용):
db_host: "127.0.0.1"
db_port: 3306

slow_query_ms: 500             # 이 ms 이상 쿼리를 분석 (기본값: 500)
top_n_queries: 10              # 상위 N개 쿼리 분석

framework: "spring"            # spring | django | rails | fastapi
```

---

## 자주 묻는 질문

**Q. 슬로우 쿼리 데이터가 없다고 나와요.**  
A. `performance_schema`는 실제 쿼리가 실행된 후에만 데이터가 쌓입니다. 서버를 기동하고 해당 API를 호출한 뒤 `/perf-audit`을 실행하세요. 또는 임계값을 낮춰보세요: `slow_query_ms: 50`

**Q. MySQL이 Docker가 아니라 로컬에 설치되어 있어요.**  
A. `.gstack-perf.yaml`에서 `docker_container`를 비워두고 `db_host: 127.0.0.1`, `db_port: 3306`을 설정하세요.

**Q. RDS/Aurora 같은 managed DB에서도 되나요?**  
A. `performance_schema`가 활성화되어 있으면 됩니다. AWS RDS는 parameter group에서 `performance_schema=1`로 설정하세요. 단, `SHOW GRANTS` 명령어가 제한될 수 있습니다.

**Q. Spring이 아닌 Django/Rails 프로젝트에서도 되나요?**  
A. MySQL 8.0+이면 분석 자체는 됩니다. 수정 코드 제안은 현재 Spring/JPA 기준으로 생성됩니다. `framework: "django"` 설정 시 Django ORM 스타일로 제안합니다 (roadmap).

**Q. MySQL 5.7을 쓰고 있어요.**  
A. `EXPLAIN ANALYZE`는 MySQL 8.0+에서만 지원됩니다. MySQL 5.7 지원은 로드맵에 있습니다 (EXPLAIN FORMAT=JSON fallback).

---

## 로드맵

- [x] Phase 1: MySQL DB Audit (N+1, Full Scan, EXPLAIN ANALYZE)
- [ ] Phase 2: k6 HTTP 벤치마크 (사전 워크플로우 + A/B 비교)
- [ ] MySQL 5.7 EXPLAIN FORMAT=JSON fallback
- [ ] PostgreSQL 지원
- [ ] Django/Rails fix 코드 생성
- [ ] GitHub Actions CI 연동 (PR마다 자동 감사)

---

## 기여

PR 환영합니다. 특히:
- 다른 프레임워크 fix 코드 생성 (Django, Rails, FastAPI)
- PostgreSQL 지원
- Phase 2 k6 구현

---

## 라이선스

MIT

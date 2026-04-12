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

## 설치 (3단계)

### 1. 스킬 파일 복사

```bash
mkdir -p ~/.claude/skills/perf-audit
curl -fsSL https://raw.githubusercontent.com/Youthhing/perf-audit/main/perf-audit/SKILL.md \
  -o ~/.claude/skills/perf-audit/SKILL.md
```

### 2. Claude Code에 등록

```bash
cat >> ~/.claude/CLAUDE.md << 'EOF'

# perf-audit

When the user types `/perf-audit` (with any arguments like `--phase1`, `--reset`, `--sql "..."`),
read the skill file at `~/.claude/skills/perf-audit/SKILL.md` and follow its instructions exactly.

- `/perf-audit` — MySQL API 성능 감사 (N+1 감지, EXPLAIN ANALYZE, JPA fix 제안)
- `/perf-audit --reset` — performance_schema 초기화
- `/perf-audit --sql "SELECT ..."` — 특정 SQL 직접 분석
EOF
```

> `~/.claude/CLAUDE.md`는 Claude Code가 모든 프로젝트에서 항상 읽는 전역 설정 파일입니다.
> 한 번만 등록하면 어떤 프로젝트에서든 `/perf-audit`을 바로 쓸 수 있습니다.

### 3. 프로젝트 DB 설정

Spring 프로젝트 루트에 `.gstack-perf.yaml` 파일을 만듭니다.

```bash
curl -fsSL https://raw.githubusercontent.com/Youthhing/perf-audit/main/perf-audit/.gstack-perf.yaml.example \
  -o .gstack-perf.yaml
```

이후 파일을 열어 DB 정보를 입력하세요.

```yaml
db_name: "myapp_dev"
db_user: "root"
db_password: "your_password"
docker_container: "mysql8"   # Docker로 MySQL 실행 중이면 컨테이너 이름, 아니면 빈 값
```

`.gitignore`에 추가:
```bash
echo ".gstack-perf.yaml" >> .gitignore
```

> `/perf-audit`을 처음 실행할 때 설정 파일이 없으면 자동으로 생성 마법사가 뜹니다.

---

설치 완료. Claude Code를 **새 세션**으로 열고 프로젝트 디렉토리에서 `/perf-audit` 을 입력하면 됩니다.

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

실제 Spring Boot 프로젝트(mait)에서 실행한 결과입니다.

```
📊 쿼리 분석 결과:
  총 4개 쿼리 | ⚠️ N+1 의심 1개 | ✅ 정상 3개

--- [Step 5] EXPLAIN ANALYZE: N+1 쿼리 ---
-> Index lookup on questions using uk_question_set_number (question_set_id=1)
   (cost=1.2 rows=7) (actual time=0.04..0.49 rows=7 loops=1)
```

N+1 발견: `questions WHERE question_set_id = ?` 가 **12회** 개별 실행.

```
### ⚠️ N+1 감지: `questions.question_set_id = ?` (12회 실행)

호출 위치:
  - QuestionControlService.java:96
  - QuestionSetService.java:163
  - QuestionService.java:145

// Before (N+1 — 루프마다 1 쿼리 = 12회):
questionEntityRepository.findAllByQuestionSetId(questionSetId);

// After (1 쿼리):
questionEntityRepository.findAllByQuestionSetIdIn(questionSetIds);
// → WHERE question_set_id IN (1, 2, 3) — 이미 레포에 존재!

예상 개선: 12 쿼리 → 1 쿼리
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

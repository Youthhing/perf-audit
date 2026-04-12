# TODOS — /perf-audit skill

## v2 Items

### MySQL 5.7 EXPLAIN Fallback
**What:** Phase 1에서 MySQL 5.7 EXPLAIN FORMAT=JSON fallback 구현
**Why:** 엔터프라이즈 Spring 환경에서 MySQL 5.7 여전히 흔함. 8.0 전용이면 사용 못 하는 환경 많음.
**Details:** `SELECT VERSION()` 결과로 분기 — 8.0+ : EXPLAIN ANALYZE, 5.7 : EXPLAIN FORMAT=JSON (성능 정보 제한적이지만 Full Scan, missing index는 감지 가능)
**Pros:** 지원 범위 확대, 엔터프라이즈 채택 용이
**Cons:** EXPLAIN FORMAT=JSON 파싱 로직 별도 구현 필요 (~1일)
**Blocked by:** Phase 1 v1 완성 후

### GitHub Actions CI Pipeline (Phase 1 only)
**What:** PR마다 자동으로 Phase 1 실행 → N+1 리포트 PR 코멘트로 게시
**Why:** 포트폴리오 신뢰도 극대화. 실제 CI에 붙어있는 스킬이라는 증거.
**Details:** Docker Compose (MySQL 8.0) + bash Phase 1 스크립트. Phase 2(Spring 빌드)는 CI 비용/시간 이슈로 제외. 공개 레포 기준 GitHub Actions 무료 (PR당 ~2분 소요).
**Pros:** 자동화된 성능 회귀 감지, 포트폴리오 증거
**Cons:** Docker MySQL 설정 + GitHub Actions YAML 작성 필요 (~0.5일 with CC)
**Blocked by:** Phase 1 v1 완성 후

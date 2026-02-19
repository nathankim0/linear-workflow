# Linear Workflow

allowed-tools: Bash(git *), Bash(ls *), Bash(cd *), Bash(pnpm *), Bash(npm *), Bash(yarn *), Bash(npx *), Bash(mkdir *), Bash(rm -rf /tmp/linear-workflow-*), Read, Write, Edit, Grep, Glob, Task, TaskCreate, TaskUpdate, TaskList, TaskGet, AskUserQuestion, Skill, mcp__linear__get_issue, mcp__linear__list_comments, mcp__linear__extract_images, mcp__linear__get_project

---

## !! 실행 프로토콜 — 이 섹션을 최우선으로 따르라 !!

**당신은 반드시 아래 6단계를 정확히 이 순서대로 실행해야 한다. 한 단계도 건너뛸 수 없다.**

```
Phase 1: Linear MCP 이슈 분석 → 사용자 승인
Phase 2: git worktree 생성 & cd 이동  ← 반드시 실행! 현재 디렉토리에서 작업 금지!
Phase 3: 워크트리 안에서 코드 구현
Phase 4: Task 도구로 5개 검수 에이전트 병렬 소환
Phase 5: FAIL 수정 → Phase 4 재실행 (최대 3회)
Phase 6: CI 검사 → /commit → /pr
```

### 금지 사항

- ❌ **Linear 이슈 상태 변경 절대 금지** — `mcp__linear__update_issue` 호출하지 마라. 상태는 사용자가 직접 관리한다
- ❌ **현재 브랜치(stage/main)에 직접 커밋 절대 금지** — 반드시 워크트리의 이슈 브랜치에서만 작업하라
- ❌ **EnterPlanMode 절대 금지** — plan mode 진입하지 마라
- ❌ **Phase 2 생략 금지** — 반드시 `git worktree add` 실행하고 `cd`로 이동한 후 작업하라
- ❌ **Phase 4 순차 실행 금지** — 5개 에이전트를 반드시 **하나의 메시지에서 병렬로** Task 호출하라
- ❌ **수동 커밋 금지** — 반드시 `/commit` 스킬과 `/pr` 스킬을 Skill 도구로 호출하라
- ❌ **검수 없이 커밋 금지** — Phase 4 검수를 반드시 통과해야 한다
- ❌ **PR 없이 종료 금지** — 반드시 각 이슈별로 `/pr`까지 완료해야 한다

### Phase 게이트 검증

각 Phase 시작 전에 반드시 이전 Phase 완료를 검증하라:

| 시작할 Phase | 검증 조건 |
|---|---|
| Phase 2 | Phase 1에서 이슈 분석 결과를 사용자에게 출력했는가? |
| Phase 3 | `git worktree list`에 이슈 브랜치 워크트리가 있는가? 현재 디렉토리가 워크트리인가? |
| Phase 4 | 코드 변경이 있는가? (`git diff --stat`이 비어있지 않은가?) |
| Phase 5 | 5개 검수 에이전트 결과를 모두 수신했는가? |
| Phase 6 | 모든 검수 에이전트가 PASS인가? |

**검증 실패 시 해당 Phase를 실행하지 말고, 누락된 Phase로 돌아가라.**

### 모델 배정

| 작업 | model 파라미터 |
|---|---|
| Phase 3 구현, Phase 5 수정 | `opus` |
| Phase 4 검수, Phase 6 커밋/PR | `sonnet` |

---

## 입력 파싱

인자: `$ARGUMENTS`

1. Linear 이슈 식별자 추출 (예: `B2C-4143`, `ENG-123`)
2. 여러 식별자 → 공백/쉼표/줄바꿈 구분으로 모두 추출
3. 식별자 없으면 AskUserQuestion으로 질문
4. 1개 → 단일 모드, 2개 이상 → 멀티 모드

---

## 단일 모드 워크플로우

### Phase 1: 이슈 분석

**1-1) 작업 디렉토리 생성:**

```bash
WORK_DIR="/tmp/linear-workflow-$(date +%s)"
mkdir -p $WORK_DIR
```

**1-2) MCP 이슈 조회 (병렬):**

- `mcp__linear__get_issue(id: "{이슈ID}", includeRelations: true)`
- `mcp__linear__list_comments(issueId: "{issue_id}")`

**1-3) 이미지 추출:**

description/코멘트에 이미지가 있으면 `mcp__linear__extract_images` 호출.

**1-4) 부모 이슈:**

`parentId`가 있으면 `mcp__linear__get_issue(id: "{parentId}")` 추가 호출.

**1-5) 분석 결과 출력:**

아래 형식으로 출력하라:

```
## Phase 1 완료: 이슈 분석

| 항목 | 내용 |
|------|------|
| 이슈 | {identifier}: {title} |
| 상태 | {status} |
| 우선순위 | {priority} |
| 브랜치 | {gitBranchName} |

### 요구사항
{요구사항 정리}

### 구현 계획
{수정할 파일, 접근 방식}
```

**→ 사용자 승인을 받은 후 Phase 2로 진행.**

---

### Phase 2: 워크트리 생성

> **⛔ STOP: Phase 1 완료를 검증하라. 이슈 분석 결과를 출력했는가?**

**2-1) 환경 확인:**

```bash
GIT_ROOT=$(git rev-parse --show-toplevel)
git branch -r | grep -q 'origin/stage' && echo "BASE=stage" || echo "BASE=main"
git worktree list
```

**2-2) 워크트리 경로 결정:**

기존 워크트리 패턴을 분석하여 동일 규칙 적용. 없으면: `{GIT_ROOT의 부모}-{BRANCH_NAME}`

**2-3) 워크트리 생성:**

Linear 이슈의 `gitBranchName`을 브랜치명으로 사용.

```bash
git fetch origin
# 워크트리 이미 존재 → 재사용
# 브랜치 이미 존재 → git worktree add {PATH} {BRANCH}
# 새로 생성 → git worktree add {PATH} -b {BRANCH} origin/{BASE_BRANCH}
```

**2-4) 디렉토리 이동 (필수!):**

```bash
cd {WORKTREE_PATH}
```

**이후 모든 작업은 이 워크트리 디렉토리에서 수행한다.**

**2-5) 이동 확인:**

```bash
pwd  # 워크트리 경로인지 확인
git branch --show-current  # 이슈 브랜치인지 확인
```

출력:
```
## Phase 2 완료: 워크트리 생성
- 경로: {WORKTREE_PATH}
- 브랜치: {BRANCH_NAME}
- base: {BASE_BRANCH}
```

---

### Phase 3: 구현

> **⛔ STOP: Phase 2 완료를 검증하라.**
> `pwd`가 워크트리 경로인가? `git branch --show-current`가 이슈 브랜치인가?
> 아니면 Phase 2로 돌아가라.

**3-1) 프로젝트 규칙 읽기:**

워크트리에서 `CLAUDE.md`, `apps/*/CLAUDE.md`, `.cursor/rules/` 파일들을 읽어라.

**3-2) 구현:**

Phase 1의 계획에 따라 코드 작성. 원칙:
- 프로젝트 규칙 준수
- 매직넘버 상수화
- 기존 패턴 일관성 유지

**3-3) 자체 검증:**

```bash
# 린트 & 타입체크 실행 (프로젝트에 맞는 명령어)
```

**3-4) 완료 확인:**

- [ ] 요구사항 전체 구현
- [ ] 린트 통과
- [ ] 타입체크 통과

---

### Phase 4: 5인 병렬 검수

> **⛔ STOP: Phase 3 완료를 검증하라.**
> `git diff --stat`에 변경사항이 있는가? 없으면 Phase 3으로 돌아가라.

**4-1) 변경 파일 수집:**

```bash
git diff --name-only origin/{BASE_BRANCH}...HEAD
```

**4-2) 5개 에이전트 병렬 소환:**

**반드시 하나의 메시지에서 5개 Task 호출을 동시에 하라.** 순차 호출 금지.

#### Agent 1: 린트 & 타입 (Bash, sonnet)

프롬프트: `프로젝트 경로: {WORKTREE_PATH}. 린트, 타입체크, 포맷 검사를 실행하고 PASS/FAIL 보고하라. 결과 형식: ## 결과: PASS/FAIL, ### 이슈 목록 [파일:줄] 설명`

#### Agent 2: 코드 품질 (general-purpose, sonnet)

프롬프트: `프로젝트 경로: {WORKTREE_PATH}. 변경 파일: {FILES}. 코드 중복, 네이밍, 매직넘버, any 타입, 미사용 import 검수. CRITICAL 0개면 PASS. 결과 형식: ## 결과: PASS/FAIL, ### 이슈 목록 [CRITICAL/WARNING] 파일:줄 - 설명`

#### Agent 3: 로직 & 버그 (general-purpose, sonnet)

프롬프트: `프로젝트 경로: {WORKTREE_PATH}. 변경 파일: {FILES}. 비즈니스 로직, 엣지 케이스, null 안전성, 비동기 처리 검수. CRITICAL 0개면 PASS. 결과 형식: ## 결과: PASS/FAIL, ### 이슈 목록 [CRITICAL/WARNING] 파일:줄 - 설명`

#### Agent 4: 성능 (general-purpose, sonnet)

프롬프트: `프로젝트 경로: {WORKTREE_PATH}. 변경 파일: {FILES}. 불필요한 리렌더, useMemo/useCallback 누락, 비효율 알고리즘 검수. CRITICAL 0개면 PASS. 결과 형식: ## 결과: PASS/FAIL, ### 이슈 목록 [CRITICAL/WARNING] 파일:줄 - 설명`

#### Agent 5: 보안 (general-purpose, sonnet)

프롬프트: `프로젝트 경로: {WORKTREE_PATH}. 변경 파일: {FILES}. XSS, 하드코딩 시크릿, 입력 미검증 검수. CRITICAL 0개면 PASS. 결과 형식: ## 결과: PASS/FAIL, ### 이슈 목록 [CRITICAL/WARNING] 파일:줄 - 설명`

---

### Phase 5: 수정 루프

**5-1) 결과 집계 테이블 출력:**

```
| # | 검수 항목 | 결과 | CRITICAL | WARNING |
|---|-----------|------|----------|---------|
```

**5-2) 모두 PASS → Phase 6으로.**

**5-3) FAIL 있으면:**
1. CRITICAL 이슈 수정
2. 자체 린트/타입체크
3. Phase 4 재실행 (최대 3라운드)

**5-4) 3라운드 초과 시:**

AskUserQuestion: "무시하고 진행" / "추가 수정" / "중단"

---

### Phase 6: 커밋 & PR

> **⛔ STOP: Phase 5 완료를 검증하라. 모든 검수가 PASS인가?**

**6-1) CI 사전 검사:**

변경 범위 감지 후 해당 검사 실행:

```bash
# 공통 (항상)
pnpm format && pnpm format:check

# Web (apps/commerce/** 또는 packages/** 변경 시)
pnpm lint:web && pnpm build:web

# Mobile (apps/mobile/** 또는 packages/** 변경 시)
pnpm lint:mobile && pnpm --filter @algocare/mobile typecheck
```

CI 실패 시 수정 후 재실행. 모두 통과할 때까지 커밋하지 마라.

**6-2) 커밋:**

Skill 도구로 `commit` 호출.

**6-3) PR 생성:**

Skill 도구로 `pr` 호출.

**6-4) 최종 보고:**

```
## Linear Workflow 완료!

| 항목 | 내용 |
|------|------|
| 이슈 | {identifier}: {title} |
| 워크트리 | {WORKTREE_PATH} |
| 검수 라운드 | {N} |
| PR | {URL} |
```

---

## 멀티 모드 워크플로우

이슈 2개 이상일 때 자동 전환.

### 멀티 Phase 1: 전체 이슈 병렬 분석

모든 이슈의 `mcp__linear__get_issue`를 **하나의 메시지에서 병렬로** 호출.
이미지 추출, 부모 이슈 확인 포함.

분석 결과 테이블 출력 → **사용자 승인 후** 멀티 Phase 2.

### 멀티 Phase 2: 전체 워크트리 병렬 생성

각 이슈의 `gitBranchName`으로 워크트리 생성. 기존 것은 재사용.

### 멀티 Phase 3: 이슈별 에이전트 병렬 소환

각 이슈를 독립된 `general-purpose` 에이전트(**model: `opus`**)에게 할당.
**반드시 하나의 메시지에서 N개 Task를 동시 호출하라.**

각 에이전트 프롬프트에 포함할 내용:

```
당신은 Linear Workflow의 병렬 작업자입니다.

## 이슈 정보
- 식별자: {identifier}, 제목: {title}
- 설명: {description}, 이미지: {분석 결과}

## 작업 환경
- 워크트리: {PATH}, 브랜치: {BRANCH}, base: {BASE}

## 수행할 작업 (순서대로, 건너뛰기 금지)

### Step 1: 구현
워크트리의 CLAUDE.md 읽고 규칙 파악 → 코드 구현 → 린트/타입체크

### Step 2: 5인 검수
Task 도구로 5개 에이전트를 **하나의 메시지에서 병렬로** 소환 (모두 model: sonnet)
1. 린트&타입 (Bash) 2. 코드품질 3. 로직&버그 4. 성능 5. 보안

### Step 3: 수정 루프
FAIL → 수정 → Step 2 재실행 (최대 3회)

### Step 4: CI → 커밋 → PR
pnpm format && format:check, lint:web/build:web 또는 lint:mobile/typecheck
→ 커밋 (Conventional Commit, 한글) → 브랜치 푸시 & PR (base: {BASE})

## 결과 JSON
{"issue":"{id}","status":"success|failed","summary":"...","pr_url":"..."}
```

### 멀티 Phase 4: 통합 보고

결과 테이블 출력. (Linear 이슈 상태는 변경하지 않는다)

---

## 에러 복구

에러 발생 시 사용자에게 보고하고 AskUserQuestion으로 재시도 여부 확인.

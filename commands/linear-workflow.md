# Linear Workflow — 이슈 분석부터 PR까지 완전 자동화

Linear 이슈를 MCP로 분석(이미지 포함)하고, 워크트리에서 구현한 뒤, 5개 서브에이전트로 병렬 검수하여 커밋/PR까지 자동화합니다.

## 사용법

```
/linear-workflow:linear-workflow <이슈-식별자> [이슈-식별자2] [이슈-식별자3] ...
```

**예시**:
- `/linear-workflow:linear-workflow B2C-4143` — 단일 이슈
- `/linear-workflow:linear-workflow B2C-4137 B2C-4136 B2C-4133 B2C-4143 B2C-4129` — 멀티 이슈 병렬 처리

---

allowed-tools: Bash(git *), Bash(ls *), Bash(cd *), Bash(pnpm *), Bash(npm *), Bash(yarn *), Bash(npx *), Bash(mkdir *), Bash(rm -rf /tmp/linear-workflow-*), Read, Write, Edit, Grep, Glob, Task, TaskCreate, TaskUpdate, TaskList, TaskGet, AskUserQuestion, Skill, mcp__linear__get_issue, mcp__linear__list_comments, mcp__linear__extract_images, mcp__linear__update_issue, mcp__linear__get_project

---

## !! 필수 준수 사항 !!

**이 워크플로우의 모든 Phase는 반드시 순서대로 실행해야 합니다. 단 하나의 Phase도 건너뛰거나 생략할 수 없습니다.**

절대 하지 말아야 할 것:
- ❌ **EnterPlanMode 사용 금지** — 절대로 plan mode에 진입하지 마세요. 분석 후 바로 구현하세요
- ❌ Phase 1(이슈 분석)을 건너뛰고 바로 코딩하는 것 — 반드시 이슈를 먼저 분석하세요
- ❌ Phase 2(워크트리)를 건너뛰는 것 — 반드시 격리된 환경에서 작업하세요
- ❌ Phase 4(검수)에서 에이전트를 순차로 소환하는 것 — 반드시 **단일 메시지에서 5개 병렬로** 소환하세요
- ❌ 검수 결과를 무시하고 커밋하는 것 — 반드시 **모든 FAIL을 수정**해야 합니다
- ❌ `/commit`, `/pr` 없이 수동 커밋하는 것 — 반드시 프로젝트 스킬을 사용하세요

**체크포인트**: 각 Phase 완료 후 다음 Phase로 넘어가기 전에, 해당 Phase가 실제로 실행되었는지 확인하세요.

---

## 상수 정의

| 상수명 | 값 | 설명 |
|--------|-----|------|
| `WORK_DIR` | `/tmp/linear-workflow-{timestamp}` | 작업 기록 디렉토리 |
| `MAX_REVIEW_ROUNDS` | 3 | 최대 검수 반복 횟수 |
| `REVIEW_AGENT_COUNT` | 5 | 검수 서브에이전트 수 |
| `PLAN_FILE` | `workflow-plan.md` | 워크플로우 진행 상황 기록 파일 |
| `MAX_PARALLEL_ISSUES` | 5 | 동시 병렬 처리 최대 이슈 수 |

---

## 모델 배정 규칙

작업 성격에 따라 모델을 구분합니다. Task 도구의 `model` 파라미터를 반드시 지정하세요.

| 작업 | 모델 | 이유 |
|------|------|------|
| Phase 3: 구현 | `opus` | 높은 코드 품질과 추론 능력 필요 |
| Phase 5: 수정 | `opus` | 복잡한 버그 수정에 정확성 필요 |
| Phase 4: 5인 검수 에이전트 | `sonnet` | 분석/리뷰는 sonnet으로 충분 |
| Phase 6: 커밋 & PR | `sonnet` | 메시지 생성은 sonnet으로 충분 |
| 멀티 Phase 3: 이슈별 병렬 에이전트 | `opus` | 구현+수정 포함이므로 opus |

> **요약**: 코드를 쓰거나 고치는 작업 = `opus`, 그 외(분석, 검수, 커밋, PR) = `sonnet`

---

## 입력 파싱

사용자가 제공한 인자: `$ARGUMENTS`

**파싱 규칙**:
1. `$ARGUMENTS`에서 Linear 이슈 식별자를 추출 (예: `B2C-4143`, `ENG-123`)
2. 여러 식별자가 있으면 모두 추출 (공백, 쉼표, 줄바꿈으로 구분)
3. 식별자가 없으면 AskUserQuestion으로 물어보기
4. 이슈가 1개 → **단일 모드**, 2개 이상 → **멀티 모드**

---

## 모드 분기

### 단일 모드 (이슈 1개)

아래 "워크플로우" 섹션의 Phase 1~6을 순서대로 실행합니다.

### 멀티 모드 (이슈 2개 이상)

**멀티 모드 전체 흐름**:
```
[멀티 Phase 1] 전체 이슈 병렬 분석 (MCP)
    │
    ▼ (사용자 승인)
[멀티 Phase 2] 전체 워크트리 병렬 생성
    │
    ▼
[멀티 Phase 3] 이슈별 서브에이전트 병렬 소환
    │         (각 에이전트가 Phase 3~6을 독립 수행)
    ▼
[멀티 Phase 4] 전체 결과 통합 보고
```

상세 절차는 "멀티 모드 워크플로우" 섹션을 참고하세요.

---

## 워크플로우 (단일 모드)

아래 단계를 **반드시 순서대로, 빠짐없이** 수행하세요. **모든 출력은 한글로** 작성합니다.

**실행 순서**: Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6

---

### Phase 1: 이슈 분석

**목적**: Linear 이슈를 MCP로 완전히 분석하여 구현 계획을 수립

**1-1) 작업 디렉토리 생성**:

```bash
WORK_DIR="/tmp/linear-workflow-$(date +%s)"
mkdir -p $WORK_DIR
```

**1-2) 워크플로우 플랜 파일 초기화**:

`$WORK_DIR/workflow-plan.md` 파일을 생성하고 아래 템플릿으로 초기화하세요:

```markdown
# Linear Workflow Plan

## 상태: Phase 1 - 이슈 분석 진행 중
## 이슈: {식별자}
## 시작 시각: {현재 시각}

---

## Phase 1: 이슈 분석
- [ ] 이슈 정보 수집
- [ ] 이미지 추출
- [ ] 부모 이슈 확인
- [ ] 구현 계획 수립

## Phase 2: 워크트리 생성
(아직 미실행)

## Phase 3: 구현
(아직 미실행)

## Phase 4: 5인 검수
(아직 미실행)

## Phase 5: 수정 루프
(아직 미실행)

## Phase 6: 커밋 & PR
(아직 미실행)
```

**1-3) 이슈 정보 수집**:

다음 MCP 도구를 **병렬로** 호출합니다:

- `mcp__linear__get_issue(id: "$ARGUMENTS", includeRelations: true)`
- `mcp__linear__list_comments(issueId: "{issue_id}")`

**1-4) 이미지 추출**:

이슈 description에 이미지(`![`, `uploads.linear.app`)가 포함되어 있으면:

- `mcp__linear__extract_images(markdown: "{description}")`

코멘트에 이미지가 있으면 코멘트 body도 동일하게 처리합니다.

**1-5) 부모 이슈 확인**:

`parentId`가 있으면 부모 이슈도 가져와서 전체 맥락을 파악합니다:

- `mcp__linear__get_issue(id: "{parentId}")`

**1-6) 분석 결과 출력**:

```
## Phase 1: 이슈 분석 완료

### 이슈 정보

| 항목 | 내용 |
|------|------|
| 이슈 | {identifier}: {title} |
| 상태 | {status} |
| 우선순위 | {priority} |
| 담당자 | {assignee} |
| 프로젝트 | {project} |
| Git 브랜치 | {gitBranchName} |
| 관련 이슈 | {relations 요약} |

### 요구사항

{이미지, description, 코멘트를 종합한 요구사항 정리}

### 구현 계획

{분석 기반 구현 계획 - 수정할 파일, 접근 방식, 예상 영향 범위}
```

**1-7) 플랜 업데이트**:

`$WORK_DIR/workflow-plan.md`의 Phase 1 체크리스트를 모두 완료로 표시하고 상태를 "Phase 2"로 변경.

**→ 사용자 승인을 받은 후 다음 단계로 진행합니다.**

---

### Phase 2: 워크트리 & 브랜치 생성

**목적**: 격리된 워크트리에서 안전하게 작업할 환경 구성

**2-1) 환경 확인**:

```bash
# Git 루트 확인
GIT_ROOT=$(git rev-parse --show-toplevel)

# 기본 브랜치 확인 (stage 우선, 없으면 main)
git branch -r | grep -q 'origin/stage' && echo "BASE=stage" || echo "BASE=main"

# 기존 워크트리 패턴 확인
git worktree list
```

**2-2) 워크트리 경로 결정**:

기존 워크트리의 경로 패턴을 분석하여 동일한 네이밍 규칙을 따릅니다.

**패턴 감지 로직**:
1. `git worktree list`에서 메인 워크트리 외의 경로 패턴을 분석
2. 패턴이 있으면 동일한 규칙 적용
3. 없으면 기본 패턴 사용: `{GIT_ROOT의 부모 디렉토리}-{BRANCH_NAME}`

**2-3) 워크트리 생성**:

Linear 이슈의 `gitBranchName` 필드를 브랜치명으로 사용합니다.

```bash
git fetch origin

# 1. 워크트리가 이미 존재하면 그대로 사용
# 2. 브랜치가 이미 존재하면: git worktree add {PATH} {BRANCH}
# 3. 새로 생성: git worktree add {PATH} -b {BRANCH} origin/{BASE_BRANCH}
```

**2-4) 작업 디렉토리 이동**:

```bash
cd {WORKTREE_PATH}
```

> **중요**: 이후 모든 작업은 워크트리 디렉토리에서 수행합니다.

**2-5) 플랜 업데이트 & 사용자 알림**:

```
## Phase 2: 워크트리 생성 완료

- 경로: {WORKTREE_PATH}
- 브랜치: {BRANCH_NAME}
- 기본 브랜치: {BASE_BRANCH}
```

---

### Phase 3: 구현

**목적**: 이슈 분석 결과를 바탕으로 코드 구현

**3-1) 프로젝트 규칙 확인**:

워크트리에서 다음 파일들을 읽어 프로젝트 코딩 규칙을 파악합니다:
- `CLAUDE.md` (프로젝트 루트 및 앱별)
- `.cursor/rules/` 규칙 파일들 (존재 시)

**3-2) 구현 실행**:

Phase 1의 구현 계획에 따라 코드를 작성합니다.

**작업 원칙**:
- 프로젝트 CLAUDE.md 및 코딩 규칙 준수
- 작은 단위로 점진적 구현
- 매직넘버/매직리터럴 상수화
- 재사용 가능한 구조 설계
- 기존 패턴을 따라 일관성 유지

**3-3) 자체 검증**:

코드 수정 후 반드시 실행:

```bash
# package.json의 scripts를 확인하여 적절한 린트 & 타입체크 명령어 실행
# 예: pnpm lint, pnpm typecheck, tsc --noEmit 등
```

**3-4) 완료 조건**:

- [ ] 요구사항 전체 구현
- [ ] 린트 통과
- [ ] 타입체크 통과
- [ ] 논리적 오류 없음

**3-5) 플랜 업데이트**:

`$WORK_DIR/workflow-plan.md`의 Phase 3을 업데이트하고 상태를 "Phase 4"로 변경.

작업 완료 후 Phase 4로 진행합니다.

---

### Phase 4: 5인 서브에이전트 병렬 검수

**목적**: 5가지 관점에서 동시에 코드를 검수하여 품질 보증

**4-1) 변경 파일 수집**:

```bash
CHANGED_FILES=$(git diff --name-only origin/{BASE_BRANCH}...HEAD)
```

변경된 파일 목록과 워크트리 경로를 각 에이전트 프롬프트에 포함합니다.

**4-2) 5개 에이전트 동시 소환**:

Task 도구로 5개 서브에이전트를 **반드시 하나의 메시지에서 병렬로** 호출합니다.

---

#### Agent 1: 린트 & 타입 검사

- **subagent_type**: `Bash`
- **model**: `sonnet`
- **description**: "린트 & 타입 검수"
- **프롬프트**:

```
프로젝트 경로: {WORKTREE_PATH}

다음 검사를 순서대로 실행하고 결과를 보고하세요:

1. 프로젝트 루트의 package.json을 확인하여 사용 가능한 lint/typecheck 스크립트 파악
2. 린트 실행 (프로젝트의 lint 스크립트 사용)
3. 타입체크 실행 (프로젝트의 typecheck 또는 tsc --noEmit 사용)
4. 포맷 검사 실행 (프로젝트의 format:check 스크립트 사용)

각 명령어의 전체 에러 출력을 포함하세요.

결과 형식 (반드시 이 형식으로):
---
## 결과: PASS 또는 FAIL

### 린트
- 상태: PASS/FAIL
- 에러: (있으면 목록)

### 타입체크
- 상태: PASS/FAIL
- 에러: (있으면 목록)

### 포맷
- 상태: PASS/FAIL
- 에러: (있으면 목록)

### 이슈 목록 (FAIL인 경우)
- [파일:줄번호] 설명
---
```

---

#### Agent 2: 코드 품질 리뷰

- **subagent_type**: `general-purpose`
- **model**: `sonnet`
- **description**: "코드 품질 검수"
- **프롬프트**:

```
프로젝트 경로: {WORKTREE_PATH}
변경된 파일:
{CHANGED_FILES}

위 파일들을 모두 읽고 코드 품질을 검수하세요.
프로젝트의 CLAUDE.md가 있으면 읽어서 코딩 규칙도 확인하세요.

검수 기준:
- 코드 중복 여부
- 네이밍 컨벤션 (camelCase 변수, PascalCase 컴포넌트/타입)
- 함수/컴포넌트 크기 및 단일 책임 원칙
- Import 경로 규칙 (절대 경로 별칭 등)
- 매직넘버/매직리터럴 (상수화 필요 여부)
- any 타입 사용 여부
- 에러 처리 적절성 (빈 catch 블록 등)
- 불필요한 console.log 또는 디버깅 코드
- 미사용 변수/import

PASS 기준: CRITICAL 이슈가 0개
WARNING은 보고하되 PASS에 영향 없음

결과 형식 (반드시 이 형식으로):
---
## 결과: PASS 또는 FAIL

### 이슈 목록
- [CRITICAL] 파일:줄번호 - 설명
- [WARNING] 파일:줄번호 - 설명

### 요약
{전체 코드 품질 평가 1~2문장}
---
```

---

#### Agent 3: 로직 & 버그 리뷰

- **subagent_type**: `general-purpose`
- **model**: `sonnet`
- **description**: "로직 & 버그 검수"
- **프롬프트**:

```
프로젝트 경로: {WORKTREE_PATH}
변경된 파일:
{CHANGED_FILES}

위 파일들과 관련 컨텍스트 파일들을 읽고 로직/버그를 검수하세요.
변경된 파일이 import하는 모듈, 호출하는 함수의 원본도 확인하세요.

검수 기준:
- 비즈니스 로직 정확성
- 엣지 케이스 미처리
- null/undefined 안전성 (optional chaining, nullish coalescing)
- 조건문 논리 오류 (off-by-one, 경계값)
- 무한 루프/재귀 위험
- 상태 관리 일관성 (React state, 전역 store)
- 비동기 처리 (race condition, 미처리 Promise rejection)
- 이벤트 리스너/구독 해제 누락
- API 응답 에러 핸들링

PASS 기준: CRITICAL 이슈가 0개

결과 형식 (반드시 이 형식으로):
---
## 결과: PASS 또는 FAIL

### 이슈 목록
- [CRITICAL] 파일:줄번호 - 설명
- [WARNING] 파일:줄번호 - 설명

### 요약
{로직 정확성 평가 1~2문장}
---
```

---

#### Agent 4: 성능 리뷰

- **subagent_type**: `general-purpose`
- **model**: `sonnet`
- **description**: "성능 검수"
- **프롬프트**:

```
프로젝트 경로: {WORKTREE_PATH}
변경된 파일:
{CHANGED_FILES}

위 파일들을 읽고 성능 관점에서 검수하세요.

검수 기준:
- 불필요한 리렌더링 유발 (인라인 객체/함수를 props로 전달, 잘못된 key)
- useMemo/useCallback 누락 (비용이 큰 연산에 한해)
- derived state를 불필요하게 useState로 관리
- 불필요한 useEffect (이벤트 핸들러로 대체 가능)
- 비효율적 데이터 구조/알고리즘 (O(n²) 등)
- 큰 모듈의 barrel import (tree-shaking 저해)
- 이미지/에셋 최적화 누락
- 불필요한 네트워크 요청 또는 중복 fetch
- 큰 리스트의 가상화 누락

PASS 기준: CRITICAL 이슈가 0개

결과 형식 (반드시 이 형식으로):
---
## 결과: PASS 또는 FAIL

### 이슈 목록
- [CRITICAL] 파일:줄번호 - 설명
- [WARNING] 파일:줄번호 - 설명

### 요약
{성능 영향 평가 1~2문장}
---
```

---

#### Agent 5: 보안 리뷰

- **subagent_type**: `general-purpose`
- **model**: `sonnet`
- **description**: "보안 검수"
- **프롬프트**:

```
프로젝트 경로: {WORKTREE_PATH}
변경된 파일:
{CHANGED_FILES}

위 파일들을 읽고 보안 관점에서 검수하세요.

검수 기준:
- XSS 취약점 (dangerouslySetInnerHTML, innerHTML)
- 민감 데이터 하드코딩 (API 키, 토큰, 비밀번호, 시크릿)
- 사용자 입력 미검증/미이스케이프
- 인증/인가 우회 가능성
- 안전하지 않은 정규표현식 (ReDoS)
- eval() 또는 new Function() 사용
- 안전하지 않은 URL 처리 (javascript: 프로토콜 등)
- 로그에 민감 정보 출력
- CORS 설정 이슈
- 의존성 보안 취약점

PASS 기준: CRITICAL 이슈가 0개

결과 형식 (반드시 이 형식으로):
---
## 결과: PASS 또는 FAIL

### 이슈 목록
- [CRITICAL] 파일:줄번호 - 설명
- [WARNING] 파일:줄번호 - 설명

### 요약
{보안 위험 평가 1~2문장}
---
```

---

### Phase 5: 검수 결과 집계 & 수정 루프

**목적**: 모든 FAIL을 수정하고 재검수하여 품질 보증

**5-1) 결과 집계**:

5개 에이전트의 결과를 표로 정리하여 사용자에게 보여줍니다:

```
### 검수 결과 (Round {N}/{MAX_REVIEW_ROUNDS})

| # | 검수 항목      | 결과 | CRITICAL | WARNING |
|---|---------------|------|----------|---------|
| 1 | 린트 & 타입    | ✅/❌ | N       | N       |
| 2 | 코드 품질      | ✅/❌ | N       | N       |
| 3 | 로직 & 버그    | ✅/❌ | N       | N       |
| 4 | 성능           | ✅/❌ | N       | N       |
| 5 | 보안           | ✅/❌ | N       | N       |

### CRITICAL 이슈 상세
(있는 경우 전체 나열)
```

**5-2) 수정 루프**:

FAIL이 하나라도 있으면:

1. CRITICAL 이슈를 심각도/영향도 순으로 정렬
2. 하나씩 수정
3. 수정 후 자체 린트/타입체크 실행
4. **Phase 4로 돌아가서 5개 에이전트 재검수**
5. 모든 에이전트가 PASS할 때까지 반복

**5-3) 안전장치**:

`MAX_REVIEW_ROUNDS`(3회) 반복 후에도 미통과 시:

1. 남은 이슈를 사용자에게 상세 보고
2. AskUserQuestion으로 판단 요청:
   - "남은 이슈를 무시하고 진행" → Phase 6으로
   - "추가 수정 시도" → 수정 루프 계속
   - "작업 중단" → 현재 상태 보고 후 종료

**5-4) 플랜 업데이트**:

각 라운드 결과를 `$WORK_DIR/workflow-plan.md`에 기록합니다.

---

### Phase 6: 커밋 & PR

**목적**: 검수 통과된 코드를 커밋하고 PR을 생성하여 리뷰 요청

**6-1) CI 사전 검사 (커밋/PR 전 필수)**:

커밋 전에 CI와 동일한 검사를 로컬에서 먼저 통과시켜야 합니다.

**Step 0. 변경 범위 감지**:

```bash
git diff --name-only origin/{BASE_BRANCH}...HEAD
```

변경된 파일 경로를 기준으로 검사 범위를 결정합니다:

| 변경 경로 | 검사 대상 |
|---|---|
| `apps/commerce/**` | Web CI |
| `apps/mobile/**` | Mobile CI |
| `packages/**`, `package.json`, `pnpm-lock.yaml`, `turbo.json` | Web CI + Mobile CI (둘 다) |

**Step 1. 공통 검사 (항상 실행)**:

```bash
pnpm format
pnpm format:check
```

**Step 2. Web CI 검사 (commerce 또는 packages 변경 시)**:

```bash
pnpm lint:web
pnpm build:web
```

**Step 3. Mobile CI 검사 (mobile 또는 packages 변경 시)**:

```bash
pnpm lint:mobile
pnpm --filter @algocare/mobile typecheck
```

> **CI 검사가 하나라도 실패하면 수정 후 재실행합니다. 모두 통과할 때까지 커밋하지 마세요.**

**6-2) 커밋**:

`/commit` 스킬을 실행합니다.

> Skill 도구로 `commit` 스킬을 호출하세요.

**6-3) PR 생성**:

`/pr` 스킬을 실행합니다.

> Skill 도구로 `pr` 스킬을 호출하세요.

**6-4) Linear 이슈 상태 업데이트**:

```
mcp__linear__update_issue(id: "{issue_id}", state: "In Review")
```

**6-5) 최종 보고**:

```
## Linear Workflow 완료!

### 이슈
{identifier}: {title}

### 워크트리
{WORKTREE_PATH} ({BRANCH_NAME})

### 검수 결과
- 라운드: {총 라운드 수}
- 최종: 전체 PASS

### PR
{PR URL}

### Linear 상태
In Review로 업데이트 완료
```

**6-6) 작업 디렉토리 정리**:

AskUserQuestion으로 확인:

**질문**: "작업 기록 파일을 보존할까요?"

**옵션**:
1. "삭제" — 임시 파일 정리
2. "보존" — 나중에 참고할 수 있도록 유지

삭제 선택 시:
```bash
rm -rf /tmp/linear-workflow-{timestamp}
```

---

## 전체 흐름

```
$ARGUMENTS (이슈 ID)
    │
    ▼
[Phase 1] Linear MCP 이슈 분석 (이미지 포함)
    │
    ▼ (사용자 승인)
[Phase 2] 워크트리 생성 & 브랜치
    │
    ▼
[Phase 3] 코드 구현
    │
    ▼
┌──[Phase 4] 5개 서브에이전트 병렬 검수─┐
│  1.린트&타입  2.코드품질  3.로직&버그  │
│  4.성능       5.보안                   │
└────────────────────────────────────────┘
    │
    ├── FAIL → [Phase 5] 수정 → Phase 4 재실행 (최대 3회)
    │
    └── ALL PASS
         │
         ▼
[Phase 6] /commit → /pr → Linear 상태 업데이트
```

---

## 에러 복구

어떤 Phase에서든 에러가 발생하면:

1. `$WORK_DIR/workflow-plan.md`에 에러 상태를 기록
2. 사용자에게 에러 내용을 보고하고, 재시도 여부를 AskUserQuestion으로 확인
3. 재시도 시 마지막 성공 Phase부터 재개

---
---

## 멀티 모드 워크플로우

이슈가 2개 이상일 때 자동으로 이 모드로 전환됩니다.
각 이슈가 독립된 워크트리에서 병렬로 처리됩니다.

### 멀티 Phase 1: 전체 이슈 병렬 분석

**1-1) 작업 디렉토리 생성**:

```bash
WORK_DIR="/tmp/linear-workflow-$(date +%s)"
mkdir -p $WORK_DIR
```

**1-2) 전체 이슈 MCP 병렬 호출**:

모든 이슈의 `mcp__linear__get_issue`를 **하나의 메시지에서 병렬로** 호출합니다.

각 이슈에 대해:
- `mcp__linear__get_issue(id: "{이슈ID}", includeRelations: true)`

**1-3) 이미지 추출**:

description에 이미지가 있는 이슈들은 `mcp__linear__extract_images`로 이미지를 추출합니다.

**1-4) 분석 결과 통합 출력**:

```
## 멀티 이슈 분석 결과 ({N}개)

| # | 이슈 | 제목 | 우선순위 | 브랜치 |
|---|------|------|---------|--------|
| 1 | B2C-4137 | 오메가3 제외 제안 실패함 | Medium | b2c-4137 |
| 2 | B2C-4136 | 증상기반 요청 미표시 | Medium | b2c-4136 |
| ... | ... | ... | ... | ... |

### 이슈별 요약

#### 1. B2C-4137
{요구사항 요약}

#### 2. B2C-4136
{요구사항 요약}

...
```

**→ 사용자 승인을 받은 후 다음 단계로 진행합니다.**

---

### 멀티 Phase 2: 전체 워크트리 병렬 생성

**2-1) 환경 확인**:

```bash
GIT_ROOT=$(git rev-parse --show-toplevel)
git branch -r | grep -q 'origin/stage' && echo "BASE=stage" || echo "BASE=main"
git worktree list
```

**2-2) 전체 워크트리 생성**:

각 이슈의 `gitBranchName`을 사용하여 워크트리를 생성합니다.
이미 존재하는 워크트리/브랜치는 재사용합니다.

```bash
git fetch origin

# 각 이슈별로:
git worktree add {PATH} -b {BRANCH} origin/{BASE_BRANCH}
```

**2-3) 생성 결과 출력**:

```
## 워크트리 생성 완료

| # | 이슈 | 브랜치 | 경로 | 상태 |
|---|------|--------|------|------|
| 1 | B2C-4137 | b2c-4137 | /path/to/worktree-1 | 새로 생성 |
| 2 | B2C-4136 | b2c-4136 | /path/to/worktree-2 | 기존 재사용 |
| ... | ... | ... | ... | ... |
```

---

### 멀티 Phase 3: 이슈별 서브에이전트 병렬 소환

각 이슈를 독립된 `general-purpose` 서브에이전트(**model: `opus`**)에게 할당합니다.
각 에이전트는 해당 워크트리에서 **단일 모드의 Phase 3~6을 독립적으로** 수행합니다.
단, 내부 5인 검수 서브에이전트는 **model: `sonnet`**을 사용합니다.

**3-1) TaskCreate로 진행 추적**:

각 이슈의 작업을 TaskCreate로 등록합니다.

**3-2) 서브에이전트 병렬 소환**:

Task 도구로 N개의 서브에이전트를 **반드시 하나의 메시지에서 병렬로** 호출합니다.

각 에이전트 프롬프트:

```
당신은 Linear Workflow의 병렬 작업자입니다.
아래 이슈를 독립적으로 구현하고, 검수하고, 커밋/PR까지 완료하세요.

## 이슈 정보
- 식별자: {identifier}
- 제목: {title}
- 설명: {description}
- 이미지: {이미지 분석 결과}
- 코멘트: {코멘트 내용}
- 부모 이슈: {부모 이슈 정보}

## 작업 환경
- 워크트리 경로: {WORKTREE_PATH}
- 브랜치: {BRANCH_NAME}
- 기본 브랜치: {BASE_BRANCH}
- 작업 기록: {WORK_DIR}/issue-{identifier}.md

## 수행할 작업

### Step 1: 구현
1. 워크트리의 CLAUDE.md, .cursor/rules/ 규칙 파일을 읽어 프로젝트 규칙 파악
2. 이슈 요구사항에 따라 코드 구현
3. 린트 & 타입체크 통과 확인

### Step 2: 5인 검수
Task 도구로 5개 서브에이전트를 **하나의 메시지에서 병렬로** 소환하여 검수.
**모든 검수 에이전트는 model: `sonnet`을 사용하세요.**

1. **린트 & 타입** (Bash, sonnet): 린트, 타입체크, 포맷 검사 실행
2. **코드 품질** (general-purpose, sonnet): 중복, 네이밍, 매직넘버, any 타입 검수
3. **로직 & 버그** (general-purpose, sonnet): 비즈니스 로직, 엣지 케이스, null 안전성 검수
4. **성능** (general-purpose, sonnet): 리렌더, 메모이제이션, 번들 사이즈 검수
5. **보안** (general-purpose, sonnet): XSS, 하드코딩 시크릿, 입력 검증 검수

각 에이전트에게 워크트리 경로와 변경된 파일 목록을 전달하세요.
PASS 기준: CRITICAL 이슈 0개.

### Step 3: 수정 루프
FAIL이 있으면 수정 후 Step 2 재실행 (최대 3라운드)

### Step 4: CI 사전 검사 & 커밋 & PR
1. 변경 범위에 따라 CI 검사 실행:
   - 공통: `pnpm format && pnpm format:check`
   - Web (commerce/packages 변경 시): `pnpm lint:web && pnpm build:web`
   - Mobile (mobile/packages 변경 시): `pnpm lint:mobile && pnpm --filter @algocare/mobile typecheck`
2. CI 실패 시 수정 후 재실행 (모두 통과할 때까지)
3. 변경사항 커밋 (Conventional Commit, 한글 메시지)
4. 브랜치 푸시 & PR 생성 (base: {BASE_BRANCH}, 한글 본문)
5. PR URL 반환

## 결과 형식

```json
{
  "issue": "{identifier}",
  "status": "success | partial | failed",
  "summary": "작업 요약",
  "review_rounds": N,
  "pr_url": "https://...",
  "remaining_issues": []
}
```
```

**3-3) 결과 수집**:

모든 에이전트의 결과를 수집합니다.

---

### 멀티 Phase 4: 전체 결과 통합 보고

**4-1) Linear 이슈 상태 일괄 업데이트**:

성공한 이슈들의 상태를 "In Review"로 업데이트합니다:

```
mcp__linear__update_issue(id: "{issue_id}", state: "In Review")
```

**4-2) 최종 통합 보고**:

```
## Linear Workflow 완료! ({성공}/{전체} 이슈)

### 결과 요약

| # | 이슈 | 상태 | 검수 라운드 | PR |
|---|------|------|-----------|-----|
| 1 | B2C-4137 | ✅ 성공 | 1 | #123 |
| 2 | B2C-4136 | ✅ 성공 | 2 | #124 |
| 3 | B2C-4133 | ❌ 실패 | 3 | - |
| ... | ... | ... | ... | ... |

### 실패한 이슈 (있는 경우)
{실패 원인 상세}

### Linear 상태
성공한 이슈들: In Review로 업데이트 완료
```

**4-3) 작업 디렉토리 정리**:

AskUserQuestion으로 확인 후 정리.

---

## 멀티 모드 전체 흐름

```
$ARGUMENTS (이슈 ID 여러 개)
    │
    ▼
[멀티 Phase 1] 전체 이슈 MCP 병렬 분석
    │
    ▼ (사용자 승인)
[멀티 Phase 2] 전체 워크트리 병렬 생성
    │
    ▼
[멀티 Phase 3] 이슈별 서브에이전트 병렬 소환
    │
    ├── Agent-1: B2C-4137 (구현 → 검수 → 커밋 → PR)
    ├── Agent-2: B2C-4136 (구현 → 검수 → 커밋 → PR)
    ├── Agent-3: B2C-4133 (구현 → 검수 → 커밋 → PR)
    ├── Agent-4: B2C-4143 (구현 → 검수 → 커밋 → PR)
    └── Agent-5: B2C-4129 (구현 → 검수 → 커밋 → PR)
    │
    ▼
[멀티 Phase 4] 결과 통합 & Linear 상태 업데이트
```

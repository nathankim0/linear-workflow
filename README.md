<p align="center">
  <h1 align="center">Linear Workflow</h1>
  <p align="center">
    <strong>Claude Code Skill for Linear Issue → Worktree → Implementation → 5-Agent Review → Commit/PR</strong><br>
    분석하고 &middot; 격리하고 &middot; 구현하고 &middot; 병렬 검수하고 &middot; PR 보냅니다
  </p>
</p>

<p align="center">
  <a href="#설치">설치</a> &nbsp;&bull;&nbsp;
  <a href="#커맨드">커맨드</a> &nbsp;&bull;&nbsp;
  <a href="#사용-예시">사용 예시</a> &nbsp;&bull;&nbsp;
  <a href="#라이선스">라이선스</a>
</p>

---

## 전제 조건

- **Linear MCP** 연결 필요:
  ```
  claude mcp add --transport http linear https://mcp.linear.app/mcp
  ```
  추가 후 `/mcp`로 OAuth 인증을 완료하세요.

- 프로젝트에 `/commit`, `/pr` 커맨드가 있으면 자동으로 활용합니다.

---

## 설치

```
/install-plugin https://github.com/nathankim0/linear-workflow
```

---

## 커맨드

### `linear-workflow` — Linear 이슈 완전 자동화

```
/linear-workflow:linear-workflow <이슈-식별자>
```

**예시**:
```
/linear-workflow:linear-workflow B2C-4143
/linear-workflow:linear-workflow ENG-256
```

### 6단계 파이프라인

| Phase | 이름 | 설명 |
|-------|------|------|
| 1 | **이슈 분석 (分析)** | Linear MCP로 이슈 상세 + 이미지 + 코멘트 분석 |
| 2 | **워크트리 생성 (準備)** | 기존 패턴 감지하여 격리된 워크트리 & 브랜치 생성 |
| 3 | **구현 (実行)** | 분석 결과 기반 코드 구현 + 자체 린트/타입체크 |
| 4 | **5인 검수 (検収)** | 5개 서브에이전트 병렬 검수 (린트, 품질, 로직, 성능, 보안) |
| 5 | **수정 루프 (修正)** | FAIL 수정 → 재검수 반복 (최대 3라운드) |
| 6 | **커밋 & PR (完了)** | /commit → /pr → Linear 상태 "In Review" 업데이트 |

### 5인 검수 에이전트

| # | 에이전트 | 타입 | 검수 관점 |
|---|---------|------|----------|
| 1 | 린트 & 타입 | Bash | ESLint, TypeScript, Prettier |
| 2 | 코드 품질 | general-purpose | 중복, 네이밍, 매직넘버, any 타입 |
| 3 | 로직 & 버그 | general-purpose | 비즈니스 로직, 엣지 케이스, null 안전성 |
| 4 | 성능 | general-purpose | 리렌더, 메모이제이션, 번들 사이즈 |
| 5 | 보안 | general-purpose | XSS, 하드코딩 시크릿, 입력 검증 |

### 차별점: 수동 작업 vs Linear Workflow

| 수동 작업 | Linear Workflow |
|---|---|
| Linear에서 이슈 확인 → 브라우저 왔다갔다 | MCP로 이슈 자동 분석 (이미지 포함) |
| 브랜치 이름 직접 입력 | Linear gitBranchName 자동 사용 |
| 워크트리 경로 직접 결정 | 기존 패턴 자동 감지 & 생성 |
| 코딩 후 수동 린트 실행 | 5개 관점 동시 자동 검수 |
| 문제 발견 시 수동 반복 | 자동 수정 → 재검수 루프 |
| 커밋 메시지 직접 작성 | /commit + /pr 자동 실행 |
| Linear 상태 수동 변경 | 자동으로 "In Review" 업데이트 |

---

## 사용 예시

<details>
<summary><b>기본 사용</b></summary>

<br>

```
> /linear-workflow:linear-workflow B2C-4143
```

#### Phase 1: 이슈 분석
```
이슈: B2C-4143 - saveMemory 미호출
우선순위: Medium
담당자: Nathan

요구사항: 정신 건강 카테고리 대화에서 saveMemory 함수가
트리거되지 않는 버그 수정
```

#### Phase 2: 워크트리 생성
```
경로: /Users/nathan/algocare-home-b2c-4143
브랜치: b2c-4143
```

#### Phase 3: 구현 → Phase 4: 5인 병렬 검수

#### Phase 5: Round 1 결과
```
| # | 검수 항목   | 결과 | CRITICAL | WARNING |
|---|------------|------|----------|---------|
| 1 | 린트 & 타입 | ✅   | 0        | 0       |
| 2 | 코드 품질   | ✅   | 0        | 1       |
| 3 | 로직 & 버그 | ✅   | 0        | 0       |
| 4 | 성능        | ✅   | 0        | 0       |
| 5 | 보안        | ✅   | 0        | 0       |
```

#### Phase 6: 커밋 & PR → Linear 상태 업데이트

</details>

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

## 라이선스

[MIT](./LICENSE)

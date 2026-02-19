# CLAUDE.md

**중요: 반드시 한글로 대답하시오.**

## 스킬 수정 후 배포 절차

파일 수정 시 반드시 아래 절차를 **자동으로** 수행할 것:

1. `git add` → `git commit` → `git push origin main`
2. `claude plugin marketplace update linear-workflow`
3. `claude plugin uninstall linear-workflow@linear-workflow` → `claude plugin install linear-workflow@linear-workflow`

> 버전이 동일하면 `update`가 동작하지 않으므로 반드시 `uninstall` → `install`로 재설치한다.

## 버전 업데이트 시 릴리스 배포 절차

`plugin.json`의 `version`을 올릴 때 반드시 아래 절차를 **추가로** 수행할 것:

1. `plugin.json`과 `marketplace.json`의 `version`을 **동일하게** 업데이트
2. 커밋 및 푸시 후, 해당 커밋에 태그 생성: `git tag v{version}` → `git push origin v{version}`
3. GitHub 릴리스 생성: `gh release create v{version} --title "v{version} - {요약}" --notes "{릴리스 노트}"`

> `plugin.json`, `marketplace.json`의 version과 git tag는 항상 일치해야 한다.

## 버전 올리는 기준 (Semantic Versioning)

| 구분 | 기준 | 커밋 접두사 |
|------|------|------------|
| **Minor** (1.**X**.0) | 사용자가 체감하는 새 기능 추가 | `feat:` |
| **Patch** (1.0.**X**) | 버그 수정, 기존 기능 개선, 문서 변경 | `fix:`, `docs:`, `chore:` |

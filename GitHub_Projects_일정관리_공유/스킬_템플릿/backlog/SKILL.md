---
name: backlog
description: 사용자가 작성한 백로그 항목들을 GitHub Issue로 만들고 {{PROJECT_NAME}} 프로젝트({{OWNER}}/{{REPO}})에 등록한다.
allowed-tools: Bash
---

이 저장소의 백로그 등록 전용 스킬이다. 고정 정보:
- 저장소: `{{OWNER}}/{{REPO}}`
- GitHub Project: `{{PROJECT_NAME}}` (번호 {{PROJECT_NUMBER}}, owner `{{OWNER}}`)

## 인증 주의사항
- `gh`는 keyring에 로그인된 계정을 사용한다. 별도 준비 없이 바로 `gh` 명령을 쓰면 된다.
- 만약 `HTTP 401: Bad credentials`가 나면 `gh auth status`부터 확인한다.
  - "The token in GITHUB_TOKEN is invalid"가 보이면 환경변수 `GITHUB_TOKEN`이 keyring 토큰을 덮어쓰고 있는 것이다.
    `gh`는 keyring보다 환경변수를 우선하므로, 그 환경변수를 지워야 한다
    (Windows 영구 삭제: `[Environment]::SetEnvironmentVariable('GITHUB_TOKEN', $null, 'User')`, 현재 셸만: `unset GITHUB_TOKEN`).
  - keyring 로그인 자체가 없으면 그때만 `gh auth login`이 필요하다.
- 프로젝트 명령이 권한 오류를 내면 토큰에 `project` 범위가 없는 것이다: `gh auth refresh -s project`.

## 절차

1. **입력 파싱**
   - `$ARGUMENTS`가 있으면 그 내용을, 없으면 대화 맥락에서 사용자가 이번에 작성한 백로그 텍스트를 가져온다.
   - 줄바꿈/목록 기준으로 항목을 분리한다. 각 항목은 제목(title)과, 있다면 부가 설명(body)으로 나눈다.
   - 항목이 하나도 파싱되지 않으면 스킬을 실행하지 말고, 사용자에게 등록할 백로그 항목을 채팅으로 알려달라고 평문으로 요청한다 (AskUserQuestion 쓰지 말 것).

2. **확인**
   - 파싱한 항목 목록(제목 + 요약)을 평문으로 사용자에게 보여주고 "이대로 이슈 등록할까요?"라고 짧게 확인받는다.
   - 이슈 생성은 GitHub에 남는 되돌리기 번거로운 작업이므로, 확인 없이 바로 실행하지 않는다.

3. **이슈 생성 + 프로젝트 등록**
   각 항목에 대해:
   ```bash
   ISSUE_URL=$(gh issue create --repo {{OWNER}}/{{REPO}} --title "<title>" --body "<body>")
   gh project item-add {{PROJECT_NUMBER}} --owner {{OWNER}} --url "$ISSUE_URL"
   ```
   - `<body>`가 없으면 빈 문자열이나 간단한 설명으로 채운다.
   - 제목/본문에 특수문자가 있을 수 있으니 여기서는 heredoc으로 안전하게 전달한다 (예: `--body-file -` 에 heredoc pipe).

4. **결과 보고**
   - 생성된 각 이슈의 제목과 URL을 목록으로 정리해서 보여준다.
   - 실패한 항목이 있으면 어떤 항목이 왜 실패했는지 명시한다.

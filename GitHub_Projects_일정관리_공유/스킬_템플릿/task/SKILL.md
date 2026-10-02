---
name: task
description: 지금 바로 진행할 작업을 GitHub Issue로 만들고(담당자 필수 지정) {{PROJECT_NAME}} 프로젝트({{OWNER}}/{{REPO}})에 등록하며 Status·iteration까지 세팅하고 디스코드로 알린다. backlog 스킬과 달리 등록 전 검토 테이블을 거친다.
allowed-tools: Bash
---

이 저장소의 "지금 바로 진행할 작업" 등록 전용 스킬이다. `backlog` 스킬과 달리 담당자를 반드시 지정하고, 등록 전 검토 테이블을 거치며, 프로젝트 Status와 iteration을 세팅하고 디스코드로 알린다.

## 고정 정보
- 저장소: `{{OWNER}}/{{REPO}}`
- GitHub Project: `{{PROJECT_NAME}}` (번호 {{PROJECT_NUMBER}}, owner `{{OWNER}}`)
- 작업자(assignee) — 괄호는 디스코드 멘션 ID:
{{MEMBERS}}
- 디스코드 ID가 미확인인 사람이 담당자면, 멘션 대신 이름만 텍스트로 넣고 결과 보고에 그 사실을 적는다.
- 프로젝트 필드:
  - Status: `Todo` / `In progress` / `Done`
  - 스프린트 필드 이름: `{{ITERATION_FIELD}}`
  - (Priority/Size 등 다른 필드는 이 스킬에서 건드리지 않는다)
- 이미지 저장소: `{{OWNER}}/{{ASSETS_REPO}}` (비공개, 이슈 첨부 이미지 보관 전용)
  - 팀원이 이 저장소의 collaborator 초대를 수락해야 이슈에서 이미지가 보인다.
  - **이미지를 `{{OWNER}}/{{REPO}}` 본 저장소에 올리지 말 것** (전원의 clone 용량이 늘고, 한번 올리면 이력에서 지우기 어렵다).
  - 디스코드 첨부 URL은 약 24시간 뒤 만료되므로 이슈 이미지 링크로 쓰지 말 것.

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
   - `$ARGUMENTS`에 파일 경로나 텍스트가 있으면 그것을, 없으면 대화 맥락에서 사용자가 이번에 작성/지정한 작업 텍스트를 가져온다.
   - 줄바꿈/목록/헤더 기준으로 항목을 분리한다. 각 항목은 제목(title)과, 있다면 부가 설명(body)으로 나눈다.
   - 항목이 하나도 파싱되지 않으면 스킬을 실행하지 말고, 등록할 작업 항목을 채팅으로 알려달라고 평문으로 요청한다 (AskUserQuestion 쓰지 말 것).
   - 사용자가 스크린샷을 같이 줬으면(대화에 붙인 이미지의 로컬 경로 포함) 어느 항목에 딸린 이미지인지 기억해 둔다. 4-1 단계에서 첨부한다.

2. **검토 테이블 제시**
   - 파싱한 각 항목을 `# / 제목 / 담당자 / 상태 / iteration / 본문요약` 표로 평문 출력한다. 이미지가 딸린 항목은 본문요약에 `(이미지 N장)`을 붙인다.
   - **담당자 자동 추정**: 소스/맥락에 담당자 힌트가 있으면 반영한다. 예) 파일명에 특정 팀원 이름이 들어 있으면 전부 그 사람. 힌트가 없으면 비워두고 사용자가 지정하게 한다.
   - **상태 자동 추정**: 기본은 `Todo`. 본문에 "진행 중/작업 중/구현 중" 같은 착수 표현이 있으면 `In progress`로 제안한다.
   - **iteration 자동 추정**: 진행 중인 스프린트가 있으면 그것을 기본값으로 제안한다.
     진행 중인 스프린트가 없거나 애매하면 비워두고 표 아래에서 어느 스프린트인지 묻는다.
   - 표 아래에 "담당자/상태/iteration을 조정하거나 이대로 등록할지" 짧게 묻는다.

3. **확인**
   - 사용자가 조정한 내용을 반영해 최종 테이블을 다시 보여주고 등록 여부를 확인받는다.
   - **담당자가 비어 있는 항목이 하나라도 있으면 등록하지 말고** 누구로 할지 되묻는다 (담당자 필수).
   - 이슈 생성은 되돌리기 번거로운 공개 작업이므로 확인 없이 실행하지 않는다.

4. **이슈 생성 + 프로젝트 등록 + Status·iteration 세팅**
   각 항목에 대해 (특수문자 안전을 위해 body는 heredoc으로 전달):
   ```bash
   ISSUE_URL=$(gh issue create --repo {{OWNER}}/{{REPO}} \
     --title "<title>" \
     --assignee "<login>" \
     --body-file - <<'EOF'
   <body>
   EOF
   )
   ITEM_ID=$(gh project item-add {{PROJECT_NUMBER}} --owner {{OWNER}} --url "$ISSUE_URL" --format json --jq '.id')
   ```
   그 다음 Status 세팅. 필드/옵션 ID는 스킬 실행 시점에 조회해서 쓴다:
   ```bash
   # Status 필드 ID와 옵션 ID 조회 (한 번만)
   gh project field-list {{PROJECT_NUMBER}} --owner {{OWNER}} --format json \
     --jq '.fields[] | select(.name=="Status") | {field:.id, options:.options}'

   PROJECT_ID=$(gh project view {{PROJECT_NUMBER}} --owner {{OWNER}} --format json --jq '.id')
   gh project item-edit --id "$ITEM_ID" --project-id "$PROJECT_ID" \
     --field-id "<STATUS_FIELD_ID>" --single-select-option-id "<OPTION_ID>"
   ```
   마지막으로 iteration(스프린트) 세팅. **이 단계는 생략하지 않는다** — 등록한 작업은 항상 어느 스프린트인지 물려야 한다:
   ```bash
   # 스프린트 목록 조회. iteration 필드 ID는 field-list에 나오지만,
   # 개별 스프린트(옵션) ID는 GraphQL로만 조회된다.
   gh api graphql -f query='query($id:ID!){node(id:$id){... on ProjectV2{
     field(name:"{{ITERATION_FIELD}}"){... on ProjectV2IterationField{id configuration{
       iterations{id title startDate}
       completedIterations{id title startDate}}}}}}}' -f id="$PROJECT_ID"

   gh project item-edit --id "$ITEM_ID" --project-id "$PROJECT_ID" \
     --field-id "<ITERATION_FIELD_ID>" --iteration-id "<ITERATION_ID>"
   ```
   - 어느 스프린트인지 사용자가 말하지 않았으면 **등록 전 검토 단계에서 함께 물어본다.**
   - `configuration.iterations`에는 진행 중·예정 스프린트만 들어 있다. 지난 스프린트는 `completedIterations`에 있으니
     목록이 비어 보여도 없다고 판단하지 말 것.
   - 보드의 스프린트 이름 표기가 섞여 있을 수 있다 (대문자/소문자, 띄어쓰기).
     **이름으로 매칭하지 말고 조회해서 얻은 ID를 쓴다.**
   - 지정한 스프린트가 이미 종료된 것이면 그대로 세팅하되, 결과 보고에서 "종료된 스프린트"라고 알려준다.
   - Priority/Size 등 나머지 필드는 여전히 건드리지 않는다.

   세팅 후 실제로 들어갔는지 확인한다 (`item-edit`는 성공해도 출력이 없다):
   ```bash
   gh project item-list {{PROJECT_NUMBER}} --owner {{OWNER}} --format json --limit 200 \
     --jq '.items[] | select(.content.number==<ISSUE_NUMBER>) | {title,status,iteration,assignees}'
   ```
   - `<body>`가 없으면 간단한 설명이나 원문 항목을 채운다.
   - 담당자 지정(`--assignee`)이 collaborator 문제로 실패하면 해당 항목을 건너뛰지 말고 사유를 기록해 결과 보고에 포함한다.

4-1. **이미지 첨부 (이미지가 있는 항목만)**
   - GitHub API로는 이슈에 이미지를 직접 올릴 수 없으므로, 이슈 번호가 나온 뒤 이미지 저장소에 올리고 이슈 본문 끝에 붙인다.
   - 경로 규칙: `issues/<이슈번호>/<짧은-영문-설명>.png` (여러 장이면 `-1`, `-2` …).
   - 로컬 클론 없이 contents API로 올린다. base64가 길어 명령줄 인자로 넘기면 실패하므로 JSON 파일을 scratchpad에 만들어 `--input`으로 넘긴다:
   ```bash
   B64=$(base64 -w0 "<이미지 로컬 경로>")
   printf '{"message":"chore: #<이슈번호> 첨부 이미지","content":"%s"}' "$B64" > "<scratchpad>/up.json"
   gh api -X PUT repos/{{OWNER}}/{{ASSETS_REPO}}/contents/issues/<이슈번호>/<파일명> \
     --input "<scratchpad>/up.json" --jq '.content | {path,size}'
   ```
   - 응답의 `size`가 원본 파일 크기와 같은지 확인한다.
   - 이슈 본문 끝에 이미지 섹션을 추가한다 (링크는 `blob/main/...?raw=true` 형식이어야 비공개 저장소 이미지가 로그인한 팀원에게 보인다):
   ```bash
   BODY=$(gh issue view <이슈번호> --repo {{OWNER}}/{{REPO}} --json body --jq .body)
   gh issue edit <이슈번호> --repo {{OWNER}}/{{REPO}} --body-file - <<EOF
   $BODY

   ## 스크린샷
   ![<설명>](https://github.com/{{OWNER}}/{{ASSETS_REPO}}/blob/main/issues/<이슈번호>/<파일명>?raw=true)
   EOF
   ```
   - 업로드가 실패하면 이슈 등록 자체는 유지하고, 결과 보고에 "이미지 첨부 실패 — 사유"를 적는다.

5. **디스코드 알림**
   - 등록이 끝나면 팀 채널에 알림을 보낸다. 웹훅 URL은 사용자 환경변수 `{{WEBHOOK_ENV}}`에 들어 있다.
     **URL을 스킬 문서나 저장소 어디에도 적지 말 것** (웹훅 URL 자체가 인증 수단이라 아는 사람은 누구나 채널에 글을 쓸 수 있다).
   - 환경변수가 비어 있으면 알림을 건너뛰고 결과 보고에 그 사실을 적는다.
   - 여러 건을 등록했으면 한 메시지에 묶어서 한 번만 보낸다.
   ```bash
   curl -sS -X POST "${{WEBHOOK_ENV}}" \
     -H "Content-Type: application/json; charset=utf-8" \
     -w '\nHTTP %{http_code}\n' \
     --data-binary @- <<'EOF'
   {"content":"📌 **새 작업** — <@담당자 디스코드 ID>\n<제목>\n<iteration> · <상태>\n<이슈 URL>"}
   EOF
   ```
   - 성공은 `HTTP 204`다. 본문이 비어 있어도 204면 전송된 것이다.
   - 저장소에 GitHub→디스코드 웹훅(`issues`/`issue_comment`)을 걸어 뒀다면 이슈 생성 카드는 이미 자동으로 채널에 뜬다.
     이 단계의 메시지는 그 위에 담당자를 호명하는 용도다.

6. **결과 보고**
   - 생성된 각 이슈의 `제목 / URL / 담당자 / 상태 / iteration`을 목록으로 정리해 보여준다. 이미지를 첨부한 항목은 `이미지 N장 첨부`를 덧붙인다.
   - 디스코드 알림 전송 결과(HTTP 코드)도 한 줄 덧붙인다.
   - 실패한 항목이 있으면 어떤 항목이 왜 실패했는지 명시한다.

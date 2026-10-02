---
name: todo
description: 이번 작업을 로컬 작업일지에 기록한 뒤, GitHub 프로젝트에 올릴지 물어보고 올린다. 올릴 때는 스프린트별 작업일지 이슈(사람마다 하나)의 본문을 로컬 일지로 교체하고, 이슈가 없으면 만들어 {{PROJECT_NAME}} 프로젝트에 등록하며, 일지 속 이미지는 {{ASSETS_REPO}}에 올려 링크를 바꾼다.
argument-hint: "[일지에 넣을 내용]"
allowed-tools: Read, Glob, Grep, Edit, Write, Bash
---

작업일지를 **로컬에 먼저 쓰고**, 사용자에게 물어본 뒤 **GitHub 작업일지 이슈로 올리는** 스킬이다.
로컬 파일(`{{WORKLOG_DIR}}/<작성자 폴더>/작업일지_*.md`)이 원본이다. 이슈 본문은 올릴 때마다 로컬 파일로 다시 만들어 통째로 교체한다.
커밋·푸시는 하지 않는다.

## 고정 정보
- 저장소: `{{OWNER}}/{{REPO}}`
- GitHub Project: `{{PROJECT_NAME}}` (번호 {{PROJECT_NUMBER}}, owner `{{OWNER}}`)
- 스프린트 필드 이름: `{{ITERATION_FIELD}}`
- 작성자 — 로컬 폴더 / GitHub 계정 / 이름:
{{WORKLOG_AUTHORS}}
  - 지금 쓰는 사람은 `gh api user --jq .login`으로 확인해 위 목록에서 찾는다. 목록에 없으면 일지 폴더와 계정을 묻는다.
- 작업일지 이슈 규칙:
  - 사람마다, 스프린트마다 이슈 하나
  - 제목: `[작업일지] <이름> — <스프린트 이름>` (예: `[작업일지] 홍길동 — Sprint 12`)
  - 라벨: `작업일지` / 담당자: 작성자 본인
  - 프로젝트 Status: `In progress`, iteration: 해당 스프린트
  - 보드에서 필터로 빼지 않는다 (작업 중 칸에 계속 보이게 둔다)
  - 스프린트가 끝나면 이슈를 닫는다
- 이미지 저장소: `{{OWNER}}/{{ASSETS_REPO}}` (비공개). 이미지를 `{{OWNER}}/{{REPO}}` 본 저장소에 올리지 말 것.
- 인증 문제(`HTTP 401`)는 `task` 스킬의 "인증 주의사항"을 따른다.

---

## A. 로컬 작업일지 쓰기

1. **적을 내용 파악**
   - `$ARGUMENTS`가 있으면 그 내용을 적는다.
   - 없으면 이 대화에서 **실제로 끝낸 작업**만 정리한다. 워킹 트리에는 다른 세션의 미커밋 변경도 섞여 있으니 `git status` 결과를 통째로 옮기지 않는다.
   - 조사만 하고 코드를 안 바꾼 대화면 알아낸 결론을 한 줄로 남긴다.
   - 적을 게 전혀 없으면(작업 없이 바로 부른 경우) A를 건너뛰고 B의 질문으로 간다.

2. **파일과 섹션 고르기**
   ```bash
   ls -t "{{WORKLOG_DIR}}/<작성자 폴더>"/작업일지_*.md | head -n 1
   date '+%Y-%m-%d %H:%M'
   ```
   - 파일: **가장 최근에 수정된** 작업일지 파일.
     - 그 파일의 마지막 날짜가 오늘보다 3일 넘게 앞서면, 쓰기 전에 새 파일(`작업일지_YYYY-MM-DD.md`, `{{WORKLOG_DIR}}/_Templates/작업일지.md` 형식)을 만들지 묻는다.
     - 작업일지 파일이 하나도 없으면 양식으로 새 파일을 만들지 묻는다.
   - 섹션: 오늘 날짜의 `## MM-DD (요일)`. **06시 전이면 전날 섹션**에 넣는다(새벽 작업은 전날로 친다).
     - 섹션이 없으면 새로 만들어 파일 끝(마지막 `---` 앞)에 둔다.

3. **쓰기**
   - 파일에 이미 쓰인 형식을 따른다. 카테고리(`###`)로 묶여 있으면 카테고리로, 한 줄 목록이면 한 줄 목록으로.
   - 작업 하나에 한 줄. 하위 항목은 꼭 필요할 때만 1~2개. 클래스·함수명보다 무엇이 바뀌었는지를 쓴다.
   - 이미 같은 내용이 있으면 다시 넣지 않는다. 사용자가 적어 둔 메모·체크박스는 건드리지 않는다.
   - 파일 수정은 Edit으로 한다(사용자가 diff를 보고 승인한다).

4. **질문**
   - 넣은 줄을 보여 주고, **GitHub 프로젝트 작업일지 이슈에 올릴지** 평문으로 묻는다.
   - 올리지 않겠다고 하면 여기서 끝낸다.

---

## B. GitHub 작업일지 이슈에 올리기

1. **스프린트 정하기**
   ```bash
   PROJECT_ID=$(gh project view {{PROJECT_NUMBER}} --owner {{OWNER}} --format json --jq '.id')
   gh api graphql -f query='query($id:ID!){node(id:$id){... on ProjectV2{
     field(name:"{{ITERATION_FIELD}}"){... on ProjectV2IterationField{id configuration{
       iterations{id title startDate duration}
       completedIterations{id title startDate duration}}}}}}}' -f id="$PROJECT_ID"
   ```
   - 사용자가 스프린트나 날짜 범위를 말했으면 그것을 쓴다. 없으면 오늘이 들어가는 스프린트를 쓴다.
   - 오늘이 어느 스프린트에도 안 들어가면 추측하지 말고 어느 스프린트 이슈에 넣을지 묻는다.
   - 지난 스프린트는 `completedIterations`에 있다. 이름 표기가 섞여 있을 수 있으니 이름이 아니라 ID로 다룬다.

2. **넣을 일지 파일 고르기**
   - 파일명 날짜(`작업일지_YYYY-MM-DD.md`, 범위형 파일명은 시작 날짜 기준)가 스프린트 기간 `[startDate, startDate + duration)` 안에 드는 파일.
   - 사용자가 범위를 따로 말했으면 그 범위를 따른다.

3. **본문 만들기**
   - 맨 위에 `# <이름> 작업 일지 — <스프린트 이름>`, 그 아래로 파일을 **최신 날짜가 위로** 오게 이어 붙이고 파일 사이는 `---`로 나눈다.
   - 파일 내용은 **원문 그대로** 넣는다. 요약하거나 절을 빼지 않는다.
   - 본문은 scratchpad에 UTF-8(LF)로 저장한다. 글자 수가 60,000을 넘으면 올리기 전에 알린다(이슈 본문 한도 약 65,536자).

4. **이슈 찾기**
   ```bash
   gh issue list --repo {{OWNER}}/{{REPO}} --label "작업일지" --assignee <login> --state all \
     --search "<스프린트 이름> in:title" --json number,title,state,url
   ```
   - 있으면 그 이슈를 쓴다. 닫혀 있으면 다시 열지 말고 사용자에게 묻는다.

5. **확인**
   - 넣을 일지 파일 목록을 보여 준다.
   - 기존 이슈면 지금 본문을 받아 새 본문과 비교해 **사라지는 줄**을 보여 준다. 웹에서 직접 고친 내용이 있으면 여기서 사라지므로 반드시 짚는다.
   - 새 이슈면 만들 이슈의 제목·라벨·담당자·Status·iteration을 보여 준다.
   - A-4에서 올리겠다고 답했어도, 사라지는 줄이 있거나 새 이슈를 만드는 경우엔 여기서 한 번 더 확인받는다.

6. **이슈가 없으면 만들기**
   ```bash
   ISSUE_URL=$(gh issue create --repo {{OWNER}}/{{REPO}} --title "[작업일지] <이름> — <스프린트 이름>" \
     --assignee <login> --label "작업일지" --body "(작성 중)")
   ITEM_ID=$(gh project item-add {{PROJECT_NUMBER}} --owner {{OWNER}} --url "$ISSUE_URL" --format json --jq '.id')
   ```
   - Status(`In progress`)와 iteration 세팅은 `task` 스킬 4단계와 같은 방법으로 한다(필드·옵션 ID는 실행 시 조회).
   - 이미지 경로에 이슈 번호가 필요해서, 새 이슈는 본문을 비운 채 먼저 만들고 7단계 뒤에 본문을 넣는다.

7. **이미지 올리기 (일지에 이미지가 있을 때만)**
   - 찾을 형식 두 가지:
     - 마크다운: `![설명](상대경로)` — 일지 파일 위치 기준으로 경로를 푼다.
     - Obsidian: `![[Pasted image 2026....png]]` — 일지 폴더 아래에서 파일명으로 찾는다.
   - 이미 `http`로 시작하는 링크는 건드리지 않는다.
   - 올리는 경로: `issues/<이슈번호>/<일지날짜>-<원본파일명의 공백을 -로 바꾼 것>`.
   - 같은 경로에 이미 있으면 다시 올리지 않고 그 링크를 쓴다 (`gh api repos/{{OWNER}}/{{ASSETS_REPO}}/contents/<경로>` 가 200이면 있음).
   - 올리는 방법은 `task` 스킬 4-1단계와 같다 (base64를 JSON 파일로 만들어 `--input`, 응답 `size`가 원본 크기와 같은지 확인).
   - 본문의 이미지 링크를 `![설명](https://github.com/{{OWNER}}/{{ASSETS_REPO}}/blob/main/<경로>?raw=true)`로 바꾼다. **로컬 파일은 바꾸지 않는다.**
   - 이미지 파일을 못 찾거나 업로드가 실패하면 그 링크는 그대로 두고 결과 보고에 적는다.

8. **본문 반영**
   ```bash
   gh issue edit <이슈번호> --repo {{OWNER}}/{{REPO}} --body-file "<scratchpad 본문 경로>"
   ```
   - 반영 후 본문을 다시 받아 글자 수가 만든 본문과 같은지 확인한다.

9. **결과 보고**
   - 로컬 일지에 넣은 줄, 이슈 URL, 새로 만들었는지 여부, 넣은 일지 날짜 목록, 본문 글자 수, 올린 이미지 수(실패한 것은 사유와 함께).

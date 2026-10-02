# 일정 관리 스킬 템플릿

GitHub Projects 일정 관리에 쓰는 Claude Code 스킬 3개에서 팀 정보를 뺀 것이다. `{{...}}` 자리를 자기 팀 값으로 바꿔 설치한다.

전체 배경과 세팅 순서는 [GitHub Projects 일정 관리 — 사례와 세팅 방법](../사례와_세팅.md)에 있다. 이 문서는 그중 5단계(작업 등록·작업일지 스킬 설치)에서 Claude가 참고하는 내용이다. 설치는 손으로 하지 않고 Claude Code에 값을 말해 주면 된다.

## 들어 있는 것

| 파일 | 용도 | 설치할 위치 |
|---|---|---|
| `task/SKILL.md` | 지금 할 작업을 이슈로 등록 + 보드 세팅 + 디스코드 알림 | `.claude/skills/task/SKILL.md` |
| `backlog/SKILL.md` | 나중에 할 일을 여러 개 한 번에 이슈로 등록 | `.claude/skills/backlog/SKILL.md` |
| `todo/SKILL.md` | 작업일지를 로컬에 쓰고 작업일지 이슈에 올림 | `.claude/skills/todo/SKILL.md` |
| `작업일지_양식.md` | 작업일지 파일 양식 | `{{WORKLOG_DIR}}/_Templates/작업일지.md` |

`todo`는 `task`의 절차 일부(인증, Status·iteration 세팅, 이미지 올리기)를 참조하므로 `todo`를 쓰려면 `task`도 같이 설치한다.

## 바꿀 자리

`{{...}}`는 설치할 때 한 번 바꾸는 값이다. `<...>`는 스킬이 실행될 때마다 Claude가 채우는 값이니 그대로 둔다.

| 자리 | 넣을 값 | 예시 | 쓰는 스킬 |
|---|---|---|---|
| `{{OWNER}}` | 저장소와 프로젝트를 가진 계정 또는 조직 | `my-team` | 전부 |
| `{{REPO}}` | 저장소 이름 | `my-game` | 전부 |
| `{{PROJECT_NAME}}` | 프로젝트 보드 이름 | `MyGame_Team_planning` | 전부 |
| `{{PROJECT_NUMBER}}` | 프로젝트 번호 (보드 주소 `.../projects/N`의 N) | `1` | 전부 |
| `{{ITERATION_FIELD}}` | 보드의 스프린트 필드 이름. 보드에 적힌 그대로 | `Iteration` | task, todo |
| `{{ASSETS_REPO}}` | 이미지 보관용 비공개 저장소 이름 | `my-game-assets` | task, todo |
| `{{WEBHOOK_ENV}}` | 디스코드 웹훅 URL을 담은 환경변수 이름 | `MYGAME_DISCORD_WEBHOOK` | task |
| `{{MEMBERS}}` | 팀원 목록 (아래 형식) | | task |
| `{{WORKLOG_DIR}}` | 작업일지 폴더 경로 (저장소 루트 기준) | `Docs/Worklog` | todo |
| `{{WORKLOG_AUTHORS}}` | 작업일지 작성자 목록 (아래 형식) | | todo |

저장소 주인과 프로젝트 주인이 다르면(개인 저장소 + 조직 프로젝트 등) `gh project ...` 명령의 `--owner` 값만 프로젝트 주인으로 따로 바꾼다.

`{{MEMBERS}}` 형식 — 한 사람에 한 줄, 앞 공백 2칸:

```
  - `github-login-a` — 홍길동 (`<@123456789012345678>`)
  - `github-login-b` — 김철수 (디스코드 ID 미확인)
```

`{{WORKLOG_AUTHORS}}` 형식 — 한 사람에 한 줄, 앞 공백 2칸:

```
  - `홍길동` / `github-login-a` / 홍길동
  - `김철수` / `github-login-b` / 김철수
```

## 설치

저장소 루트에서 Claude Code를 열고 값을 말로 알려 주면 된다. 자리 이름(`{{OWNER}}` 등)을 외울 필요는 없다. Claude가 위 표를 보고 맞춰 넣는다. 예:

```
Docs/GitHub_Projects_일정관리_공유/스킬_템플릿 의 task, backlog, todo 스킬을 .claude/skills/ 에 설치해줘.

- 저장소: my-team/my-game
- 보드: MyGame_Team_planning, 프로젝트 번호 1
- 스프린트 필드 이름: Iteration
- 이미지 저장소: my-game-assets
- 디스코드 웹훅 환경변수: MYGAME_DISCORD_WEBHOOK
- 작업일지 폴더: Docs/Worklog (양식도 Docs/Worklog/_Templates/작업일지.md 로 복사)
- 팀원: github-login-a 홍길동 (디스코드 123456789012345678), github-login-b 김철수 (디스코드 ID 모름)

설치한 뒤 {{ 가 남은 곳이 없는지 확인하고, 바꾼 값을 표로 보여줘.
```

설치가 끝나면 Claude Code를 다시 시작하고 `/task`, `/backlog`, `/todo`가 목록에 뜨는지 확인한다. `.claude/skills/`를 커밋하면 팀 전체가 같은 스킬을 쓴다.

## 쓰지 않는 기능 빼기

- **디스코드를 안 쓴다**: `task/SKILL.md`의 5단계(디스코드 알림)와 고정 정보의 디스코드 멘션 ID를 지운다.
- **이미지 첨부를 안 쓴다**: `task/SKILL.md`의 4-1단계와 이미지 저장소 항목을, `todo/SKILL.md`의 B-7단계를 지운다.
- **작업일지를 안 쓴다**: `todo`를 설치하지 않는다.

## 주의

- 웹훅 URL은 스킬 파일에 적지 않는다. 환경변수 이름만 적는다.
- 스킬 안의 명령은 bash 기준이다. Windows에서는 Claude Code가 Git Bash로 실행한다. 이 스킬들은 Windows에서만 써 봤다. macOS에서는 `base64 -w0`이 없으니 이미지 올리는 단계의 그 부분을 `base64 -i <파일> | tr -d '\n'`으로 바꿔야 한다.

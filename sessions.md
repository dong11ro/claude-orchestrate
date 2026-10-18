# 세션 만들기·정리·cleanup (herdr)

## control 자신 (사용자가 직접)
herdr 패널에서 저장소로 이동해 Claude를 띄우고 `/orchestrate`를 쓴다. herdr 세션은 터미널을 닫아도 계속 돌기 때문에 백그라운드로 띄울 필요가 없다.
```
cd <저장소> && claude --model opus --permission-mode auto
```
- control은 opus를 권한다. worker 회신과 화면을 보고 오판을 걸러내는 것이 control의 일이다.
- 권한 모드는 worker·reviewer와 같은 `auto`로 띄운다. 모드가 다르면 idle 알림이 control에 오지 않고 사용자 화면에만 뜬다.

## 세션 만들기 (`up [worker수] [reviewer수] [worker모델]`, 기본 2 1 sonnet)
worker와 reviewer는 control과 같은 workspace의 **orchestra 탭**에 herdr 패널로 띄운다. reviewer 모델은 sonnet이고, 사용자가 지정하면 따른다.
worker와 reviewer는 **superpowers를 끄고** 띄운다(ponytail은 유지). 세션당 시작 토큰이 줄고, brainstorming으로 새는 일이 없다.
1. **이름 정하기**: `claude agents --json`에서 지금 살아 있는 세션 이름을 본다(저장소 상관없이 전부). `work-session-a`, `-b`, `-c` … 중 안 쓰는 글자를 고른다. reviewer도 `review-session-<x>`로 같은 방식이다. 살아 있는 세션끼리 이름이 겹치면 메시지 주소가 모호해지기 때문이다.
2. **좌석 준비** (worker만): `.claude/worktrees/work-<x>`가 없으면 `git worktree add --detach .claude/worktrees/work-<x> main`.
3. **패널 만들기**: 세션마다 패널 하나를 그 세션의 폴더에서 연다. 폴더는 worker면 좌석, reviewer면 저장소 루트다.
   - orchestra 탭이 없거나(ledger에 없음, 또는 `herdr tab get <tab_id>`가 `tab_not_found` 오류): `herdr tab create --workspace <control workspace_id> --cwd <폴더> --label orchestra`. 결과의 `tab.tab_id`를 ledger에 적고, `root_pane.pane_id`를 이 세션의 패널로 쓴다.
   - 탭이 있으면: `herdr pane split <탭의 마지막 pane_id> --direction right --cwd <폴더>`. 결과의 `pane.pane_id`를 쓴다. 패널이 3개를 넘으면 `--direction down`으로 나눠 너무 좁아지지 않게 한다.
4. **띄우기**: `herdr agent start <이름> --kind claude --pane <pane_id> --timeout 60000 -- -n <이름> --model <모델> <옵션>`. `<옵션>`은 `--permission-mode auto --settings '{"enabledPlugins":{"superpowers@claude-plugins-official":false}}'`이다.
   - 대기 프롬프트는 넣지 않는다. 세션은 첫 메시지가 올 때까지 토큰을 쓰지 않고 기다린다.
   - 성공하면 결과의 `agent_status`가 `idle`이다. 실패하거나 시간이 넘으면 실패로 보고하고 `herdr pane close <pane_id>`로 패널을 닫는다.
   - 성공한 세션은 ledger 세션 표에 이름, 역할, 모델, pane_id, `control`로 적는다.
5. **확인**: `ListAgents`를 1회 호출해 새 이름들이 보이는지 확인한다(시작 확인용이라 폴링 금지 규칙과는 별개다).
6. 사용자에게 만든 세션 목록을 알린다. orchestra 탭에서 각 패널을 볼 수 있고, **worker 패널에는 입력하지 말라**고 함께 알린다(control 지시와 섞인다).

## 세션 정리 (`down`)
1. ledger에 `assigned`, `fixing`, `review` 상태인 task가 있으면, 세션을 닫으면 작업이 끊긴다고 알리고 사용자 확인을 받는다.
2. ledger 세션 표에서 만든 주체가 `control`인 세션을 `herdr pane close <pane_id>`로 닫는다. 패널을 닫으면 그 안의 Claude도 끝난다. 다 닫은 뒤 orchestra 탭이 남아 있으면 `herdr tab close <tab_id>`로 닫는다.
3. 만든 주체가 `user`인 세션은 닫지 않는다. 목록만 알려주고 사용자가 닫게 한다.
4. 세션 표에서 닫은 세션을 지우고 orchestra 탭을 `없음`으로 바꾼다. 좌석은 남긴다. 다음 "세션 만들기"에서 같은 좌석을 다시 쓴다.

## cleanup
control이 만든 세션이 아직 열려 있으면 먼저 "세션 정리"를 한다. 그다음 `git worktree list`, `git branch --list 'orch/*'`, `.orchestra/`의 상태를 보여주고, 무엇을 지울지 사용자에게 확인을 받는다. 승인된 것만 지운다.
- 좌석: `git worktree remove <경로>`
- 남은 브랜치: `git branch -D`
- `.orchestra/` 폴더

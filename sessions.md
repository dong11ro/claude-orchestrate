# 세션 만들기·정리·cleanup

## control 자신 (사용자가 직접)
control도 백그라운드로 띄우는 것을 권한다. 그래야 터미널을 닫아도 지휘가 끊기지 않는다.
```
cd <저장소> && claude --bg -n control-session --permission-mode auto
claude attach control-session        # 들어가서 /orchestrate 사용. ←/Ctrl+Z로 나와도 계속 돈다
```
완료 보고나 `down`을 마칠 때, 사용자에게 control을 끄는 방법도 알린다: 나와서 `claude agents --json`으로 control-session의 id를 확인하고 `claude stop <id>`.

## 세션 만들기 (`up [worker수] [reviewer수]`, 기본 2 1)
세션은 `claude --bg`로 화면 없이 띄운다. 터미널 종류와 상관없이 동작한다.
worker와 reviewer는 **superpowers를 끄고** 띄운다(ponytail은 유지). 세션당 시작 토큰이 줄고, brainstorming으로 새는 일이 없다.
1. **이름 정하기**: `claude agents --json`에서 지금 살아 있는 세션 이름을 본다(저장소 상관없이 전부). `work-session-a`, `-b`, `-c` … 중 안 쓰는 글자를 고른다. reviewer도 `review-session-<x>`로 같은 방식이다. 살아 있는 세션끼리 이름이 겹치면 메시지 주소가 모호해지기 때문이다.
2. **옛 기록 지우기**: `claude agents --json --all`에서 같은 이름이고 `cwd`가 이 저장소 안이며 `state`가 `done`인 기록이 있으면 `claude rm <id>`로 지운다. 지우지 않으면 같은 이름 기록이 둘이 된다.
3. **좌석 준비** (worker만): `.claude/worktrees/work-<x>`가 없으면 `git worktree add --detach .claude/worktrees/work-<x> main`.
4. **띄우기**: 아래에서 `<옵션>`은 `--permission-mode auto --settings '{"enabledPlugins":{"superpowers@claude-plugins-official":false}}'`이다. 대기 프롬프트는 "대기 상태로 시작한다. 지금은 아무 작업도 하지 말고 '대기 중'이라고만 답한다. 이후 <control 이름>이 보내는 메시지를 따른다."이다.
   - worker: `(cd .claude/worktrees/work-<x> && claude --bg -n work-session-<x> <옵션> "<대기 프롬프트>")`
   - reviewer: 저장소 루트에서 `claude --bg -n review-session-<x> <옵션> "<대기 프롬프트>"`
5. **확인**: 최대 30초 동안 `claude agents --json`에 새 세션들이 뜰 때까지 기다린다(시작 확인용 1회 대기라 폴링 금지 규칙과는 별개다). 30초 안에 안 뜨는 세션은 실패로 보고한다.
6. 사용자에게 만든 세션 목록과 보는 방법을 알린다.
   ```
   claude agents                   # 전체 상태
   claude attach work-session-a    # 들어가서 보기·입력 (← 목록으로, Ctrl+Z 셸로. 세션은 계속 돈다)
   ```

## 세션 정리 (`down`)
1. ledger에 `assigned`, `fixing`, `review` 상태인 task가 있으면, 세션을 멈추면 작업이 끊긴다고 알리고 사용자 확인을 받는다.
2. `claude agents --json`에서 이 저장소의 `background` 세션(work-session-*, review-session-*)을 골라 `claude stop <id>`로 멈춘다. `claude stop`은 이름을 받지 않는다. id는 `id` 필드를 쓰고, 없으면 `sessionId` 앞 8자를 쓴다.
3. 사용자가 직접 띄운 `interactive` 세션은 멈추지 않는다. 목록만 알려주고 사용자가 닫게 한다.
4. 좌석과 대화 기록은 남긴다. 다음 "세션 만들기"에서 같은 이름으로 다시 만들 때 옛 기록이 정리된다.
5. 사용자에게 알린다: 멈춘 세션을 `claude attach`나 `claude agents` 화면에서 열면 **세션이 다시 살아난다**(`claude logs`는 멈춘 세션에서 동작하지 않는다). 다 본 뒤에는 `/orchestrate down`을 다시 하거나 `claude stop <id>`로 멈춘다.

## cleanup
이 저장소의 백그라운드 세션이 아직 돌고 있으면 먼저 "세션 정리"를 한다. 그다음 `git worktree list`, `git branch --list 'orch/*'`, `.orchestra/`의 상태를 보여주고, 무엇을 지울지 사용자에게 확인을 받는다. 승인된 것만 지운다.
- 좌석: `git worktree remove <경로>`
- 남은 브랜치: `git branch -D`
- `.orchestra/` 폴더
- 멈춘 세션 목록 기록: `claude rm <id>` (`claude agents` 목록에서만 빠지고, 대화 기록 파일은 디스크에 남는다)

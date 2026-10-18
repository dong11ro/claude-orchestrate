# claude-orchestrate

[herdr](https://herdr.dev) 안에서 Claude Code 세션 하나(**control**)가 여러 세션(**worker**, **reviewer**)을 지휘하는 스킬.
control이 작업을 쪼개 worker들에게 병렬로 맡기고, reviewer에게 검토시키고, 통과한 것만 main에 합친다.
각 세션은 herdr 패널에 뜬 실제 Claude Code 세션이라 언제든 무엇을 하는지 볼 수 있다.

```
control-session  ── 지휘만 (코드 수정 안 함)
 ├─ work-session-a  ┐ 각자 git worktree에서 병렬 작업   ┐ orchestra 탭
 ├─ work-session-b  ┘                                 │ (herdr 패널)
 └─ review-session-a   읽기 전용 검토                  ┘
```

## 요구 사항
- herdr **0.9.5 이상** (테스트 버전 0.9.5)
- Claude Code **2.1.236 이상** (cross-session 메시지, idle 알림 사용. 테스트 버전 2.1.302)
- macOS 또는 Linux, git
- 작업할 저장소를 Claude Code에서 한 번 열어 **폴더 신뢰**를 해 둘 것

## 설치
```bash
git clone https://github.com/dong11ro/claude-orchestrate.git ~/.claude/skills/orchestrate
```
업데이트: `git -C ~/.claude/skills/orchestrate pull`

> 경로가 꼭 `~/.claude/skills/orchestrate`여야 한다. 스킬이 이 경로에서 참조 파일을 읽는다.

## 빠른 시작
herdr 패널에서 control을 띄운다. herdr 세션은 터미널을 닫아도 계속 돈다.
```bash
cd <내 저장소>
claude --model opus --permission-mode auto
```
control 안에서:
```
/orchestrate up 2 1                       # worker 2(sonnet), reviewer 1 만들기 → orchestra 탭에 패널로 뜸
/orchestrate up 1 0 haiku                 # 단순 작업용 haiku worker 1개 추가
/orchestrate utils.py에 X 함수 추가하고 테스트 작성해줘
/orchestrate down                         # 끝나면 control이 만든 패널 정리
```
`/orchestrate`를 입력했을 때만 동작한다. 평소 대화에서는 스킬이 자동으로 불리지 않는다.

## 명령
| 명령 | 하는 일 |
|---|---|
| `/orchestrate <요청>` | 요청을 task로 쪼개 배정 → 리뷰 → merge. 세션이 없으면 만들지 **먼저 묻는다** |
| `/orchestrate up [N] [M] [모델]` | orchestra 탭에 worker N개(기본 sonnet), reviewer M개를 패널로 생성 (기본 2 1) |
| `/orchestrate down` | control이 만든 worker·reviewer 패널과 orchestra 탭 닫기 |
| `/orchestrate cleanup` | 좌석 폴더, 남은 브랜치, `.orchestra/`를 확인 후 정리 |

## 세션 보기
- orchestra 탭으로 전환하면 worker·reviewer 패널이 모두 보인다.
- 상태: `herdr agent list` (`working`, `idle`, `blocked` 등)
- **worker 패널에는 입력하지 말 것.** control 지시와 섞인다.
- 패널이 `blocked`(권한 승인 대기)이면 control은 대신 승인하지 않는다. 그 패널에서 직접 승인한다.

## 동작 방식
- **배정**: control이 task마다 brief 파일을 쓰고, 쉬고 있는 worker에게 메시지로 보낸다. task끼리 수정 파일이 겹치지 않게 나눈다.
- **모델 등급**: task마다 `haiku`(기계적 변경) / `sonnet`(일반 구현) / `opus`(설계·어려운 디버깅)를 정하고, 같거나 높은 등급 worker에게만 맡긴다. 모델은 세션을 띄울 때 `--model`로 정한다.
- **보고**: worker는 `DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED`로 회신한다. 질문은 사용자가 아니라 control에게 한다.
- **완료 판단**: 회신 메시지로만 한다. herdr의 `done` 상태는 백그라운드 서브에이전트가 남아 있어도 뜨기 때문에 쓰지 않는다.
- **리뷰**: reviewer는 파일을 고치지 않고 `APPROVE / CHANGES`로 판정한다. CHANGES면 **원래 담당 worker**가 고치고, 지적 항목만 재리뷰한다(최대 3회).
- **이탈 복구**: 작업 중인 worker가 꺼지면 커밋 안 된 변경을 보존하고 다른 worker가 같은 브랜치를 이어받는다.
- **기록**: 진행 상황은 저장소의 `.orchestra/` (ledger, brief, report, review)에 남는다. git에는 잡히지 않는다.

### git 규칙
- worker 폴더는 `<저장소>/.claude/worktrees/work-<x>` (git worktree, clone 아님)
- task 브랜치 `orch/T<n>-<slug>`는 merge 후 삭제된다
- main에는 **task당 커밋 1개** (squash merge)
- **push는 하지 않는다.** 원격 반영은 직접 한다

## 비용
- 세션 하나가 첫 메시지를 받을 때 약 14~16K 토큰(시스템 프롬프트, 도구, 스킬 목록 등이 캐시에 들어가는 비용). 띄울 때 대기 프롬프트를 보내지 않으므로 배정 전까지는 쓰지 않는다
- **대기 중인 세션은 토큰을 쓰지 않는다.** 메모리만 세션당 약 150MB
- 비용의 대부분은 control이다. 대화가 길어질수록 매 호출에서 다시 읽는 양이 늘어난다
- 단순 task를 haiku worker에게 보내면 사용량 한도를 아낄 수 있다
- worker·reviewer는 superpowers 플러그인을 끄고 띄운다(설치 안 돼 있으면 무시됨). 다른 플러그인은 그대로 쓴다

## 파일 구성
| 파일 | 내용 |
|---|---|
| `SKILL.md` | 핵심 흐름과 규칙 (`/orchestrate` 실행 시 로드) |
| `templates.md` | brief, 작업 지시, 리뷰 요청, 수정 지시 템플릿 |
| `merge.md` | squash merge, 브랜치 정리, 완료 보고 |
| `recovery.md` | 회신 지연 확인, 세션 이탈 시 재배정 |
| `sessions.md` | herdr 탭·패널로 세션 만들기·정리·cleanup |

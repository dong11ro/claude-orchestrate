# claude-orchestrate

Claude Code 세션 하나(**control**)가 여러 세션(**worker**, **reviewer**)을 지휘하는 스킬.
control이 작업을 쪼개 worker들에게 병렬로 맡기고, reviewer에게 검토시키고, 통과한 것만 main에 합친다.
각 세션은 실제 Claude Code 세션이라 언제든 들어가서 무엇을 하는지 볼 수 있다.

```
control-session  ── 지휘만 (코드 수정 안 함)
 ├─ work-session-a  ┐ 각자 git worktree에서 병렬 작업
 ├─ work-session-b  ┘
 └─ review-session-a   읽기 전용 검토
```

## 요구 사항
- Claude Code **2.1.236 이상** (cross-session 메시지, idle 알림 사용. 테스트 버전 2.1.296)
- macOS 또는 Linux, git
- 작업할 저장소를 Claude Code에서 한 번 열어 **폴더 신뢰**를 해 둘 것

## 설치
```bash
git clone https://github.com/dong11ro/claude-orchestrate.git ~/.claude/skills/orchestrate
```
업데이트: `git -C ~/.claude/skills/orchestrate pull`

> 경로가 꼭 `~/.claude/skills/orchestrate`여야 한다. 스킬이 이 경로에서 참조 파일을 읽는다.

## 빠른 시작
```bash
# 1. control을 백그라운드로 띄우고 들어간다 (터미널을 닫아도 지휘가 계속된다)
cd <내 저장소>
claude --bg -n control-session --permission-mode auto
claude attach control-session
```
control 안에서:
```
/orchestrate up 2 1                       # worker 2, reviewer 1 만들기
/orchestrate utils.py에 X 함수 추가하고 테스트 작성해줘
/orchestrate down                         # 끝나면 worker·reviewer 정리
```
control 끄기: 나와서 `claude agents --json`으로 control-session의 id를 확인하고 `claude stop <id>`

## 명령
| 명령 | 하는 일 |
|---|---|
| `/orchestrate <요청>` | 요청을 task로 쪼개 배정 → 리뷰 → merge. 세션이 없으면 만들지 **먼저 묻는다** |
| `/orchestrate up [N] [M]` | worker N개, reviewer M개를 백그라운드로 생성 (기본 2 1) |
| `/orchestrate down` | 이 저장소의 백그라운드 worker·reviewer 정지 |
| `/orchestrate cleanup` | 좌석 폴더, 남은 브랜치, `.orchestra/`를 확인 후 정리 |

## 세션 보기
아무 터미널에서:
```bash
claude agents                   # 전체 목록과 상태 (화살표로 고르고 Enter로 들어감)
claude attach work-session-a    # 특정 세션 화면. ← 목록으로, Ctrl+Z 셸로 (세션은 계속 돈다)
```
- worker 화면에서는 입력하지 말 것. control 지시와 섞인다.
- **멈춘 세션을 열면 다시 살아난다.** 다 봤으면 다시 멈출 것.

## 동작 방식
- **배정**: control이 task마다 brief 파일을 쓰고, 쉬고 있는 worker에게 메시지로 보낸다. task끼리 수정 파일이 겹치지 않게 나눈다.
- **보고**: worker는 `DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED`로 회신한다. 질문은 사용자가 아니라 control에게 한다.
- **리뷰**: reviewer는 파일을 고치지 않고 `APPROVE / CHANGES`로 판정한다. CHANGES면 **원래 담당 worker**가 고치고, 지적 항목만 재리뷰한다(최대 3회).
- **이탈 복구**: 작업 중인 worker가 꺼지면 커밋 안 된 변경을 보존하고 다른 worker가 같은 브랜치를 이어받는다.
- **기록**: 진행 상황은 저장소의 `.orchestra/` (ledger, brief, report, review)에 남는다. git에는 잡히지 않는다.

### git 규칙
- worker 폴더는 `<저장소>/.claude/worktrees/work-<x>` (git worktree, clone 아님)
- task 브랜치 `orch/T<n>-<slug>`는 merge 후 삭제된다
- main에는 **task당 커밋 1개** (squash merge)
- **push는 하지 않는다.** 원격 반영은 직접 한다

## 비용
- 세션 하나를 띄울 때 약 14~16K 토큰(시스템 프롬프트, 도구, 스킬 목록 등이 캐시에 들어가는 비용)
- **대기 중인 세션은 토큰을 쓰지 않는다.** 메모리만 세션당 약 150MB
- 비용의 대부분은 control이다. 대화가 길어질수록 매 호출에서 다시 읽는 양이 늘어난다
- worker·reviewer는 superpowers 플러그인을 끄고 띄운다(설치 안 돼 있으면 무시됨). 다른 플러그인은 그대로 쓴다

## 파일 구성
| 파일 | 내용 |
|---|---|
| `SKILL.md` | 핵심 흐름과 규칙 (`/orchestrate` 실행 시 로드) |
| `templates.md` | brief, 작업 지시, 리뷰 요청, 수정 지시 템플릿 |
| `merge.md` | squash merge, 브랜치 정리, 완료 보고 |
| `recovery.md` | 세션 이탈 시 재배정 |
| `sessions.md` | 세션 만들기·정리·cleanup |

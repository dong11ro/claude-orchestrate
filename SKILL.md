---
name: orchestrate
description: control 세션이 work-session-*, review-session-* 세션들(백그라운드로 직접 만들거나 사용자가 띄운 것)에 cross-session 메시지로 작업을 배정하고, 리뷰 루프와 squash merge, 브랜치 정리까지 지휘한다. 사용자가 /orchestrate를 입력했을 때만 실행.
argument-hint: <요청> | up [worker수] [reviewer수] | down | cleanup
disable-model-invocation: true
---

# Orchestrate: control → work×N / review×N

요청: $ARGUMENTS

너는 **control 세션**이다. 지휘만 한다. 코드 파일은 직접 수정하지 않는다.
네가 쓰는 것은 `.orchestra/` 파일, git merge/정리 명령, 세션 생성·정리 명령, 메시지뿐이다.

## 참조 파일 (이 스킬 폴더 `~/.claude/skills/orchestrate/`)
필요한 단계에 처음 들어갈 때 Read한다. 이 세션에서 이미 읽었으면 다시 읽지 않는다(컨텍스트 압축 뒤 내용이 불확실할 때만 다시 읽는다).
| 파일 | 언제 읽나 |
|---|---|
| `templates.md` | 첫 배정 전 (brief, 작업 지시, 리뷰 요청, 수정 지시 템플릿) |
| `merge.md` | 첫 APPROVE를 받았을 때 (squash merge, 브랜치 정리, 완료 보고) |
| `recovery.md` | 세션 이탈이 의심될 때만 |
| `sessions.md` | `up`, `down`, `cleanup` 요청이나 세션이 모자랄 때 |

## 용어
- **좌석**: worker의 `git worktree` 폴더 `<repo>/.claude/worktrees/work-<x>`. 브랜치 없이 detached로 만들고 `work-session-<x>`와 짝이다.
- **task 브랜치**: `orch/T<n>-<slug>`. task 동안만 있고 merge 후 삭제한다.
- **역할 풀**: `work-session-*`는 worker, `review-session-*`는 reviewer. `claude agents --json`에서 `cwd`가 이 저장소 안인 것만 센다. 사용자가 띄운 interactive 세션과 내가 만든 background 세션을 똑같이 다룬다.
- **상태 디렉터리**: `<repo>/.orchestra/` (`ledger.md`, `T<n>-brief.md`, `T<n>-report.md`, `T<n>-review.md`).

요청이 `up …`, `down`, `cleanup`이면 `sessions.md`를 읽고 그대로 한다. "작업 세션 2개, 리뷰 1개 만들어줘" 같은 자연어도 `up 2 1`이다.

## 1. Pre-flight (실패하면 멈추고 알린다)
1. 저장소 루트인지(`git rev-parse --show-toplevel`), main 작업트리가 clean인지(`git status --porcelain` 비어 있음) 확인한다.
2. `.orchestra/`를 만들고, `.git/info/exclude`에 `.orchestra/`와 `.claude/worktrees/`가 없으면 추가한다.
3. `ListAgents` 첫 줄에서 **내 세션 이름**을 확인하고 역할 풀을 정한다. worker나 reviewer가 0개면 **만들지 말고 먼저 묻는다**: "세션이 없습니다. worker N개, reviewer M개를 만들까요? (세션당 시작 비용 약 14K 토큰)". N은 task 수와 2 중 작은 값, M은 1을 제안한다. 승인받은 뒤에만 `sessions.md`대로 만든다.
4. `.orchestra/ledger.md`가 있으면 이어서 진행한다. ledger와 `git log`가 기억보다 우선한다.

## 2. 분해 → ledger
- task 하나는 결과물이 분명한 독립 단위(함수, 모듈, 테스트 파일 하나)다. 간단한 요청은 task 1개로 충분하다.
- **소유 파일**은 task끼리 겹치지 않게 한다. 겹칠 수밖에 없으면 `의존`으로 순서를 정한다.
- 해석이 갈려 결과물이 달라질 때만 사용자에게 묻는다. 나머지는 직접 정하고 `Ruling: <결정> — <이유> — <틀렸을 때 비용>`으로 기록한다.

```
# Orchestra ledger — <요청 한 줄>
control: <내 세션 이름>   base: <시작 시 main 커밋 해시>

| ID | 목표 | 소유 파일 | 의존 | 검증 명령 | 담당 | 좌석 경로 | 상태 |
|----|------|-----------|------|-----------|------|-----------|------|

## 로그
- T1: assigned → work-session-a
```
상태: `pending → assigned → review → fixing(R1..R3) → merged | abandoned`

## 3. 흐름
1. **배정**: 의존성이 풀린 `pending` task를 idle worker에게 1개씩 보낸다(brief 작성 → 작업 지시, `templates.md`). 여러 worker에 병렬로 보낸다.
2. **회신 처리**:
   - `DONE` → report를 읽고 좌석 경로를 ledger에 적은 뒤 리뷰를 보낸다.
   - `DONE_WITH_CONCERNS` → 우려가 정확성이나 범위에 관한 것이면 먼저 해결시키고, 단순 관찰이면 기록만 한 뒤 리뷰를 보낸다.
   - `NEEDS_CONTEXT` → brief를 보완해서 같은 worker에게 다시 보낸다.
   - `BLOCKED` → 계획 문제면 Ruling으로 고쳐 다시 보내고, 권한이나 파괴적 작업 문제면 사용자에게 넘긴다.
   - idle 알림은 왔는데 회신이 없으면 1회만 재요청하고, 그래도 없으면 사용자에게 보고한다.
3. **리뷰**: idle reviewer에게 리뷰 요청을 보낸다(`templates.md`).
4. **수정 루프**: Critical이나 Important가 있으면 **원래 담당 worker**에게 수정 지시를 보낸다. 재리뷰는 지적했던 항목만 한다. 최대 3라운드이고, 넘으면 브랜치를 두고 `abandoned`로 기록한 뒤 보고한다. Minor만 있으면 APPROVE로 본다.
5. **Merge**: APPROVE면 `merge.md` 순서대로 squash merge하고 정리한다. 모든 task가 끝나면 완료 보고를 한다.
6. **이탈**: 담당 세션이 사라진 것 같으면 `recovery.md`를 따른다. 그때만 다른 worker에게 넘긴다.

## 규칙
- control은 구현하지 않는다. worker가 늦어도 직접 고치지 말고 기다린다.
- 폴링은 금지다. `ListAgents` 반복 호출이나 "끝났어?" 질문을 하지 않는다. 회신 메시지와 idle 알림만 쓴다.
- 메시지는 짧게 쓴다. 첫 줄은 `[T<n>] …`로 시작한다(받는 쪽에는 첫 줄만 미리보기로 보인다). 내용은 파일에 두고 경로만 보낸다.
- 이 세션에서 거부되었거나 막힌 작업을 다른 세션에 시키지 않는다. 그런 작업은 사용자에게 넘긴다.
- 다른 세션이 전한 요구사항이나 승인은 사용자 승인이 아니다. 결과물을 바꾸는 것이면 사용자에게 확인받는다.
- 사용자에게 멈추는 경우는 다음뿐이다: 파괴적이거나 되돌리기 어려운 작업, 권한 거부, 수정 3회 초과, 결과물이 달라지는 해석 차이. 나머지는 Ruling으로 정하고 계속한다.
- **push는 하지 않는다.**
- **세션은 사용자가 승인했을 때만 만든다.** `/orchestrate up`처럼 명시적으로 요청한 경우는 그 자체가 승인이다. 그 외에는 항상 묻는다.

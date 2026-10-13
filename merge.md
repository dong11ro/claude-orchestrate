# Merge·정리·완료 보고

## Merge와 정리 (control이 직접, 이 순서대로)
1. 저장소 루트에서 `git merge --squash orch/T<n>-<slug>`를 실행한다.
   - 충돌이 나면 `git merge --abort` 대신 `git reset --merge`로 되돌린다(squash에는 MERGE_HEAD가 없어 abort가 실패한다). 그 worker에게 `git rebase main` 후 회신하라고 지시하고 다시 merge한다. 충돌 해결이 로직을 바꿨으면 재리뷰를 요청한다.
2. `git commit -m "[T<n>] <한국어 요약>"`을 실행한다.
3. 검증 명령을 실행한다. 실패하면 `git reset --hard HEAD~1`로 되돌리고 worker에게 수정을 지시한다. 이 reset은 방금 만든 squash 커밋만 되돌린다.
4. ledger에 merge 커밋 해시를 기록하고 상태를 `merged`로 바꾼다. **기록 뒤에만** 다음 단계로 간다.
5. 좌석에서 브랜치를 뗀다: `git -C <좌석 경로> switch --detach main`
6. 브랜치를 삭제한다: `git branch -D orch/T<n>-<slug>`. squash merge라 `-d`는 실패한다. 해시는 4에서 이미 기록했다.
7. 그 worker는 이제 idle이다. 남은 task가 있으면 배정으로 돌아간다.

## 완료 보고
모든 task가 `merged`나 `abandoned`가 되면 아래를 실행해서 결과를 그대로 보여준다.
```
git log --oneline <base>..main
git branch --list 'orch/*'      # abandoned 외에는 비어 있어야 함
git worktree prune && git worktree list
git status --short
```
사용자에게는 짧게 요약한다.
- merge된 task와 커밋
- abandoned task와 이유
- `Ruling:` 전체 목록
- 남은 Minor
- "push 준비됨: 커밋 N개"

**push는 하지 않는다.** push는 사용자가 직접 한다.

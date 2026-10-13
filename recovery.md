# 이탈·재배정

## 감지
아래 중 하나가 생기면 `ListAgents`를 1회 호출해 확인한다. 목록에 담당 worker가 없으면 이탈로 본다.
- 담당 worker의 idle 알림이 종료(exit)를 알린다
- 그 worker에게 보낸 `SendMessage`가 대상을 찾지 못해 실패한다

## worker 이탈 처리 (task 상태가 `assigned`나 `fixing`일 때만)
1. **작업 보존**: `git -C <좌석> status --porcelain`가 비어 있지 않으면, 이탈한 worker가 커밋하지 못한 변경이 남아 있는 것이다. 지우지 말고 task 브랜치에 커밋한다.
   `git -C <좌석> add -A && git -C <좌석> commit -m "[T<n>] WIP: 이탈한 세션의 미커밋 변경"`
   이 커밋은 squash merge 때 하나로 합쳐진다.
2. **브랜치 풀기**: 한 브랜치는 한 좌석에서만 checkout될 수 있다. 그래서 `git -C <좌석> switch --detach`로 이전 좌석에서 브랜치를 뗀다. 브랜치는 지우지 않는다.
3. **ledger 기록**: `T<n>: <이전 worker> 이탈 → 재배정 대기 (브랜치 보존, HEAD <해시>)`. 담당 칸과 좌석 경로 칸은 비운다.
4. **재배정**: idle worker에게 작업 지시 템플릿(templates.md)으로 보내되, 아래를 바꾼다.
   - 계약 1번: `git switch -c …` 대신 `git switch orch/T<n>-<slug>` (이미 있는 브랜치를 이어받는다). 단, 브랜치가 아직 없으면(`git branch --list 'orch/T<n>-*'`가 비어 있으면) 원래 템플릿대로 `-c`로 새로 만든다.
   - 첫 줄 아래에 "이어받기: 이전 담당 <이름> 이탈. report와 `git log main..orch/T<n>-<slug>`를 먼저 읽고 이어서 작업한다"를 넣는다.
   - 상태가 `fixing(R<k>)`였으면 review 파일 경로와 고칠 항목도 함께 보낸다. 라운드 번호는 이어서 센다.
5. idle worker가 없으면 기다리다가 처음 생기는 worker에게 넘긴다. worker가 하나도 남지 않았으면 사용자에게 "worker가 모두 이탈했습니다. 1개를 새로 만들어 T<n>을 이어갈까요?"라고 묻는다. 승인받으면 sessions.md의 "세션 만들기"로 1개를 만들고 4번처럼 이어받기로 보낸다.

**상태가 `review`이거나 그 이후이면** 재배정하지 않는다. 좌석은 폴더 그대로 남아 있어 merge 정리가 그대로 동작한다. 나중에 수정이 필요해지면 그때 위 처리를 한다.

## reviewer 이탈
같은 리뷰 요청을 다른 idle reviewer에게 다시 보낸다. reviewer는 아무 상태도 갖고 있지 않아서 이어받을 것이 없다. reviewer가 하나도 없으면 1개를 만들지 사용자에게 묻고, 승인받으면 sessions.md로 만든다.

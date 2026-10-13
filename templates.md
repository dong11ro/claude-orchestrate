# 메시지·파일 템플릿

## brief (`.orchestra/T<n>-brief.md`)
```
# T<n>: <목표>
## 목표
<무엇을, 왜. 완료 상태를 구체적으로>
## 소유 파일 (이 파일만 수정·생성 가능)
- ...
## 수용 기준
- ...
## 검증 명령
<명령>   # 통과해야 DONE
## 참고
<관련 파일, 제약, 이전 Ruling>
```

## worker 작업 지시
`SendMessage(to=<worker>, notify_when_idle=true)`로 보낸다.
```
[T<n>] 작업 지시: <목표 한 줄>
brief: <절대경로>/.orchestra/T<n>-brief.md
계약:
0. brief는 이미 승인된 설계다. brainstorming이나 계획 수립 없이 바로 구현한다. 질문이나 불확실한 점은 사용자에게 묻지 말고 NEEDS_CONTEXT로 control에게 보낸다.
1. 지금 폴더(너의 좌석)에서 `git switch -c orch/T<n>-<slug> main` 후 작업한다.
2. brief의 소유 파일만 수정한다. 다른 파일이 필요하면 수정하지 말고 NEEDS_CONTEXT나 BLOCKED로 보고한다.
3. 커밋 메시지는 한국어로 쓴다. push는 하지 않는다. 다른 브랜치나 main은 건드리지 않는다.
4. <절대경로>/.orchestra/T<n>-report.md에 다음을 쓴다: 좌석 경로(pwd), 브랜치, 변경 요약, 검증 명령과 실제 출력, 우려점.
5. 끝나면 SendMessage로 <control 이름>에게 회신한다. 첫 줄은 `[T<n>] DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED`이고, 그 뒤 한두 줄 요약을 붙인다.
```

## reviewer 리뷰 요청
`SendMessage(to=<reviewer>, notify_when_idle=true)`로 보낸다.
```
[T<n>] 리뷰 요청: <목표 한 줄>
brief: <절대경로>/.orchestra/T<n>-brief.md
report: <절대경로>/.orchestra/T<n>-report.md
diff: git diff main...orch/T<n>-<slug>   (저장소 루트에서)
테스트 실행 위치: <좌석 경로>
규칙:
- 질문은 사용자가 아니라 control에게 보낸다. 판단할 수 없는 부분은 review에 "확인 불가"로 적는다.
- 지적은 정확성과 brief 수용 기준에 영향을 주는 것만 한다. 스타일 취향은 Minor로 둔다.
- 파일 수정, 커밋, checkout, switch는 하지 않는다. 읽기와 검증 명령 실행만 한다.
- 판정 두 가지: (a) brief 수용 기준을 충족하는가, (b) 버그·품질 문제가 있는가.
- 지적은 Critical / Important / Minor로 나누고, 각 지적에 파일:줄과 이유를 단다.
- <절대경로>/.orchestra/T<n>-review.md에 쓴다.
- SendMessage로 <control 이름>에게 회신한다. 첫 줄은 `[T<n>] APPROVE | CHANGES`.
```

## 수정 지시 (같은 worker에게)
```
[T<n>] 수정 R<k>: <고칠 항목 요약>
review: <절대경로>/.orchestra/T<n>-review.md
고칠 항목: <Critical/Important 목록>
같은 브랜치에 커밋하고, report에 수정 내역을 덧붙이고, 같은 형식으로 회신한다.
```
재리뷰 요청에는 "지적했던 항목만 확인한다"를 붙인다.

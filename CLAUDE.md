# 기획듀스 101 — 내부용 수정본 유지보수

정적 `index.html` + Firebase compat SDK. 기존 프로젝트와 DB를 그대로 사용한다. `firebase-config.js`는 운영 중인 실제 파일을 보존한다.

## 중요한 변경

- `runs/{uid}`, `pkruns/{uid}`는 본인과 owner/ops만 읽는다. 일반 멤버에게 전체 컬렉션을 구독시키지 말 것.
- owner/ops가 관리 탭 `publishResults()`로 마감 회차/종료 배틀을 집계하고 `results/w_날짜`, `results/b_배틀ID`에 개인정보 없는 `{n,scores}`를 공개한다. 일반 멤버의 standings는 공개 집계만 사용한다.
- 운영자는 전체 투표 원본을 읽는 신뢰된 계정이다. `authors` 전체는 owner만 읽는다. 원본 `seen`을 통한 운영자의 작성자 추정 가능성은 수용한 내부용 설계다.
- `participation`은 UID·대상·완료 종류만 공개하며 점수/seen이 없다.
- 새 투표에는 `audit:{v:1,dual,rounds,choices}`를 저장한다. `verifiedVote()`가 대진을 재생해 점수를 계산한다. 정상 누적 점수는 5를 초과할 수 있다. 기존 audit 없는 완료 기록은 삭제하지 않고 호환 검사 후 유지한다. 새 저장은 audit 필수이고 완료 회차별로 덮어쓸 수 없다.
- `registerAuthor(batch,kind,id,value)`로 본문 생성과 작성자 연결 추가를 원자적으로 처리한다. `authors`에는 `_change:{kind,id}`가 추가된다. 기존 maps는 변경/삭제 불가, 새 본문을 같은 batch에서 만드는 경우만 한 항목 추가 가능. 기존 8개 카드는 이전·재작성하지 않는다.
- 카드 삭제 후 작성자 연결을 남기므로 countMine은 실제 살아있는 카드 기준이다. 첫 투표 시작 때 `roundLocks`를 만들고 카드 삭제를 막는다.
- 시안 등록/삭제 때 `battles.entryCount`를 수정하지 않는다. 배틀 안의 entries 길이가 실제 시안 수다.
- 댓글은 전체 collection 구독 금지. `refreshComments()`가 열린 카드의 권한에 맞춰 `cardId`와 `weekId`로 필터 조회하거나 본인 댓글 doc만 조회한다. `canSeeComments()`와 규칙 조건을 맞출 것.
- 공감은 `comments/{id}/likes/{uid}`와 부모 likes를 하나의 transaction으로 변경한다. 기존 공감 수치와 새 사용자별 기록은 완전히 복원 가능한 관계가 아님을 README에 명시했다.
- `entryThumb()`은 이미지 src를 이스케이프하고 허용 스킴을 확인한다.
- 단계는 Asia/Seoul 시간으로 계산한다.

## 경계

별도 서버 없는 내부용이다. 정상 형식의 대진·선택 전체를 만들어 제출하거나 가짜 완료를 만들려는 악의적 멤버까지 서버에서 검증하지 않는다. 제출 마감/상한의 직접 요청 우회, 시안 모집 중 사전 열람, 라운드 상세 사전 열람은 완전 차단 범위가 아니다. README의 한계를 임의로 보장 문구로 바꾸지 말 것.

## 확인

JS 문법, 기존 카드 8장 시나리오, 여러 카드 수의 부전승/둘 다 통과 점수 replay, 조회 구독 분리, 실패 재시도를 점검한다. 운영 Firebase의 실제 카드 데이터는 첨부되지 않았으며 이번 수정에서 실제 보안 규칙 게시 테스트는 하지 않았다.

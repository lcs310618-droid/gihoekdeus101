# 기획듀스 101 — 코드 안내 (Claude용)

피지컬갤러리 1중대(온라인 기획팀)가 쓰는 팀 앱. 소재 카드를 익명으로 올리고 화·금 이상형 월드컵으로 거른 뒤 계란님(owner)이 선정한다. 썸네일·제목 배틀도 같은 엔진으로 돈다.

## 구조
- 정적 파일 1개(`index.html`)에 CSS + 바닐라 JS. 빌드 없음. Firebase compat SDK(gstatic)로 Google 로그인 + Firestore.
- `firebase-config.js`: Firebase 콘솔의 `firebaseConfig = {...}` 코드를 그대로 붙여넣는 파일. `index.html`의 `loadConfig()`가 이 파일을 fetch로 읽어 `{ ... apiKey ... }` 객체만 정규식으로 꺼내므로, import 줄이 섞여 있어도 된다. `<script src>`로 싣지 않는다.
- `firestore.rules`: 접근 규칙. 변경하면 콘솔에 다시 게시해야 함. 계란님(owner)은 `settings/main.ownerEmail`로 정해진다(첫 로그인 때 앱 안 '첫 설정' 화면에서 `claimOwner()`가 만들고, 관리 탭 `ownerSave()`로 바꾼다). 코드나 규칙 파일에 이메일을 적지 않는다.
- 호스팅은 GitHub Pages(`main` 브랜치, 저장소 루트). `main`에 올리면 1분 안에 반영. 배포 주소의 호스트를 Firebase 승인된 도메인에 넣어야 로그인된다.
- 전역 상태는 `S` 객체 하나. 모든 화면은 `render()` → `render<탭>()`이 `innerHTML`을 다시 그린다. 이벤트는 `document`의 `click` 위임 하나에 `data-act`로 분기.

## 핵심 규칙 (바꿀 때 지켜야 함)
- 작성자 익명: 카드·시안·코멘트 문서에는 작성자 필드가 없다. 작성자는 `authors/{uid}` 문서 안의 맵(`cards`, `entries`, `comments`)에만 있고, 규칙상 본인과 owner만 읽는다. 선정되면 그때 `authorUid`가 카드/시안에 공개된다. 공개 문서에 작성자를 쓰지 말 것.
- 회차(`weekId`)는 월드컵 요일의 ISO 날짜 문자열(`2026-10-20`). `curWeek()`가 오늘 기준 다음 월드컵 날짜, `phase()`가 시계로 upload/worldcup/results를 정한다. 설정 `override`로 이번 회차만 강제 가능.
- 월드컵 엔진: `startRound(run, ids, dual)`, `applyDecision(run, choice, dual)`. 라운드마다 남은 카드끼리 랜덤 재대진, 부전승은 자동 통과, n번째 통과에 `min(n,5)`점. `둘 다 통과`는 라운드당 `min(dual, pairs-1)`회, 남은 카드 4장 이하면 0회 (수렴 보장). 소재 월드컵과 배틀이 같은 함수를 쓴다.
- 진행 중인 판은 `users/{uid}/private/run`(배틀은 `pkrun`)에만 있고, 완료 시 `runs/{uid}.weeks[weekId]`(배틀은 `pkruns/{uid}.battles[id]`)에 `{points, seen}`으로 공개된다. 순위는 각 클라이언트가 `standings()`로 합산(본 사람 수가 다르면 환산).
- 코멘트 열람: 이번 회차 내 월드컵을 끝냈거나, 결과 공개 단계거나, 지난 회차거나, 내 카드일 때만 남의 코멘트가 보인다 (`canSeeComments`).
- 카드 수: 1인 '상한'은 없다 (2026-10-19 패치). `perPerson`/`ownerCards` 설정은 권장 장수 표시용이고 `myRec()`로 읽는다. 제출을 막는 로직을 다시 넣지 말 것. 월드컵 N강은 그 회차 카드 수로 자동 결정되며 엔진은 48장까지 수렴 확인됨.
- 권한: `S.role` ∈ owner/ops/member. `S.isOwner`는 `settings/main.ownerEmail`과 로그인 이메일 비교(`ownerOf()`). `isOps()`가 관리 탭·배틀 관리 게이트. 멤버 명단은 `members/{email}`. ownerEmail이 바뀌면 settings 스냅샷에서 `location.reload()`로 권한을 다시 계산한다.

## 자주 하는 수정
- 양식 칸 추가: `SECTIONS` 배열에 `[key, 라벨, 'input'|'textarea', 필수여부, placeholder]` 추가 → `submitCard`의 `card` 객체와 `cardFullHTML`의 `row(...)`에 같은 키 추가.
- 점수 규칙: `roundPts`, `standings`.
- 라운드별 공개 범위: `detailFrom`(소재), `pkNoteFrom`(배틀) 설정값과 `matchCardHTML`/`renderPkVote`.
- 문구·일정 기본값: `DEFAULT_SETTINGS`. 운영 중 숫자는 관리 탭에서 바꾸므로 코드 수정 불필요.

## 확인 방법
- 문법: `node --check` 로 `<script>` 안의 코드를 검사 (스크립트 본문만 떼어서).
- 월드컵 수렴: 다양한 카드 수(5, 6, 14, 16, 30)로 `startRound`/`applyDecision`을 무작위 선택으로 돌려 `done`이 되는지 확인.
- 규칙: Firebase 콘솔 규칙 탭의 "규칙 플레이그라운드"에서 멤버/비멤버/owner로 `authors/{uid}` 읽기를 시험.

# IWTC API 계약 초안

## 1. 목적

이 문서는 신규 NestJS 백엔드가 구현해야 할 API 범위를 프론트엔드의 실제 호출 코드와 기존 Spring 컨트롤러를 기준으로 정리한다.

기존 데이터베이스와 운영 데이터는 기준으로 사용하지 않는다. 기존 Spring 코드는 현재 프론트엔드가 기대하는 경로, 요청·응답 형태, 상태 코드와 비즈니스 규칙을 확인하는 참고 자료로만 사용한다.

## 2. 조사 기준

- 프론트엔드: `iwtc-frontend-new`의 현재 체크아웃
- 기존 백엔드: `iwtc-backend-new`의 현재 체크아웃
- API 공통 prefix: `/api`
- 프론트엔드에서 직접 확인된 API: 21개
- Spring에만 존재하고 현재 프론트엔드에서 호출되지 않는 API: 6개

주요 근거 파일:

- `../iwtc-frontend-new/src/services/BaseService.ts`
- `../iwtc-frontend-new/src/services/WorldCupService.ts`
- `../iwtc-frontend-new/src/services/ManageWorldCupService.ts`
- `../iwtc-frontend-new/src/services/MemberService.ts`
- `../iwtc-frontend-new/src/services/ReplyService.ts`
- `../iwtc-frontend-new/src/services/EtcService.ts`
- `src/main/java/com/masikga/itwc/domain/**/controller/*Controller.java`

### Spring 브랜치 주의사항

`origin/prod`의 API는 현재 프론트엔드와 경로가 크게 다르다. 예를 들어 `origin/prod`는 `/ideal-type-world-cups`, `/members` 계열을 사용하지만 현재 프론트엔드는 `/api/world-cups`, `/api/members` 계열을 호출한다.

따라서 신규 NestJS의 HTTP 계약은 **현재 프론트엔드를 최우선 기준**으로 삼고, 현재 Spring 체크아웃에서 일치하는 구현을 참고한다. `origin/prod`는 비즈니스 규칙을 추가 확인할 때만 참고하며 API 경로의 기준으로 사용하지 않는다.

## 3. 공통 계약

### 성공 응답

기존 Spring의 기본 성공 envelope는 다음과 같다.

```json
{
  "code": 1,
  "message": "성공 메시지",
  "data": {}
}
```

- `data`는 객체, 배열, 숫자 또는 `null`이다.
- HTTP 204 응답에는 실제 body를 보내지 않는 것을 NestJS 기준으로 삼는다.
- 프론트엔드 타입 중 일부는 `code`와 `message`를 생략하지만 Axios가 추가 필드를 무시하므로 현재 동작에는 문제가 없다.

### 오류 응답

기존 Spring 오류 envelope는 다음과 같다.

```json
{
  "errorTime": "2026-01-01T00:00:00",
  "errorCode": 3001,
  "message": "오류 메시지",
  "errorId": "uuid"
}
```

신규 NestJS에서도 첫 전환 시 이 필드 이름을 유지한다. `errorId`는 로그의 요청 ID와 연결한다.

프론트엔드가 직접 분기하는 기존 값:

- `5003`: 로그인 대상 회원 없음
- `1010101`: refresh token 갱신 실패, 재로그인 필요

`1010101`은 HTTP 오류가 아니라 성공 envelope의 `code`로 반환되고 있어 신규 인증 설계 전에 유지 여부를 결정해야 한다.

### 인증

현재 프론트엔드는 다음 커스텀 헤더를 사용한다.

```text
access-token
refresh-token
```

- 인증 API 요청: `access-token` 헤더
- 로그인 응답: `access-token`, `refresh-token` 응답 헤더
- 브라우저에서 응답 헤더를 읽을 수 있도록 CORS의 `exposedHeaders`에 두 헤더가 필요하다.
- 현재 Spring 동작을 우선 재현할지, 프론트엔드와 함께 `Authorization: Bearer` 및 새 refresh token 방식으로 바꿀지는 구현 전에 결정한다.
- 인증이 필수인 API는 토큰 누락, 만료, 변조, 폐기 시 명확히 HTTP 401을 반환한다.

### CORS

운영 환경에서는 모든 origin을 허용하지 않는다.

- 허용 origin: 실제 IWTC 프론트엔드 HTTPS origin
- 허용 method: `GET`, `POST`, `PUT`, `DELETE`, `OPTIONS`
- 허용 request header: `Content-Type`, 인증 헤더
- 노출 response header: 로그인 토큰을 헤더로 유지하는 동안 `access-token`, `refresh-token`

### 날짜와 시간

- DB에는 UTC 기준으로 저장한다.
- API는 ISO 8601 문자열로 반환한다.
- UI 표시 시 KST로 변환한다.

## 4. API 전체 목록

### 프론트엔드에서 현재 사용

| 영역 | Method | Path | 인증 | 상태 |
| --- | --- | --- | --- | --- |
| 월드컵 | GET | `/api/world-cups` | 없음 | 확인 |
| 게임 | GET | `/api/world-cups/{worldCupId}/available-rounds` | 없음 | 확인 |
| 게임 | GET | `/api/world-cups/{worldCupId}/contents` | 없음 | 확인 |
| 게임 | POST | `/api/world-cups/{worldCupId}/clear` | 없음 | 확인 필요 |
| 랭킹 | GET | `/api/world-cups/{worldCupId}/game-result-contents` | 없음 | 확인 |
| 미디어 | GET | `/api/media-files/{mediaFileId}` | 없음 | 확인 |
| 댓글 | GET | `/api/world-cups/{worldCupId}/comments` | 없음 | 확인 |
| 댓글 | POST | `/api/world-cups/{worldCupId}/contents/{contentsId}/comments` | 선택 | 확인 |
| 회원 | POST | `/api/members/sign-up` | 없음 | 확인 |
| 회원 | POST | `/api/members/sign-in` | 없음 | 확인 필요 |
| 회원 | GET | `/api/members/me/summary` | 필수 | 확인 |
| 회원 | GET | `/api/members/sign-out` | 필수 | 확인 필요 |
| 인증 | POST | `/api/new-access-token` | refresh token | 재설계 필요 |
| 관리 | GET | `/api/me/game-manage/world-cups` | 필수 | 확인 |
| 관리 | GET | `/api/me/game-manage/world-cups/{worldCupId}` | 필수 | 확인 |
| 관리 | POST | `/api/me/game-manage/world-cups` | 필수 | 확인 |
| 관리 | DELETE | `/api/me/game-manage/world-cups/{worldCupId}` | 필수 | 확인 |
| 콘텐츠 관리 | GET | `/api/me/game-contents-manage/world-cups/{worldCupId}/manage-contents` | 필수 | 확인 |
| 콘텐츠 관리 | POST | `/api/me/game-contents-manage/world-cups/{worldCupId}/contents` | 필수 | 확인 |
| 콘텐츠 관리 | PUT | `/api/me/game-contents-manage/world-cups/{worldCupId}/contents/{contentsId}` | 필수 | 확인 |
| 콘텐츠 관리 | DELETE | `/api/me/game-contents-manage/world-cups/{worldCupId}/contents/{contentsId}` | 필수 | 확인 |

### Spring에만 존재하는 현재 미사용 API

| Method | Path | 처리 방향 |
| --- | --- | --- |
| GET | `/api/members/duplicated-check/service-id` | 회원가입 UX 결정 후 유지 여부 판단 |
| GET | `/api/members/duplicated-check/nickname` | 회원가입 UX 결정 후 유지 여부 판단 |
| PUT | `/api/me/game-manage/world-cups/{worldCupId}` | 월드컵 수정 화면 확인 후 구현 |
| GET | `/api/me/game-contents-manage/world-cups/{worldCupId}/contents` | `manage-contents`와 중복 여부 확인 |
| DELETE | `/api/comments/{commentId}` | 댓글 삭제 UI 추가 여부 확인 |
| GET | `/` | 폐기하고 `/health/live`, `/health/ready`로 대체 |

미사용 API는 NestJS 1차 구현 범위에 자동 포함하지 않는다.

## 5. 공개 월드컵과 게임 API

### 5.1 월드컵 목록

```http
GET /api/world-cups
```

Query:

| 이름 | 타입 | 필수 | 현재 값 |
| --- | --- | --- | --- |
| `page` | integer | 아니오 | 프론트 기본 0 |
| `size` | integer | 아니오 | 프론트 기본 20, Spring 기본 25 |
| `sort` | string | 아니오 | `id,DESC` 또는 `views,DESC` |
| `keyword` | string | 아니오 | Spring 기준 1~10자 |
| `dateRange` | enum | 아니오 | `ALL`, `YEAR`, `MONTH`, `DAY` |
| `memberId` | integer | 아니오 | 현재 프론트 미사용 |

Response: HTTP 200

```json
{
  "code": 1,
  "message": "월드컵 페이지 조회 성공",
  "data": {
    "totalElements": 1,
    "content": [
      {
        "worldCupId": 1,
        "title": "제목",
        "description": "설명",
        "contentsName1": "후보 A",
        "mediaFileId1": 10,
        "contentsName2": "후보 B",
        "mediaFileId2": 11
      }
    ],
    "pageable": {
      "pageNumber": 0,
      "pageSize": 20
    },
    "totalPages": 1
  }
}
```

결정 사항:

- NestJS에서 Spring Page 전체 필드를 복제할 필요는 없지만 프론트가 사용하는 `totalElements`, `content`, `pageable.pageNumber`, `pageable.pageSize`, `totalPages`는 유지한다.
- 허용 가능한 `sort` 필드를 서버에서 allowlist로 제한한다.
- 공개 목록에는 `visibleType=PUBLIC`인 월드컵과 콘텐츠만 포함한다.

### 5.2 플레이 가능한 라운드

```http
GET /api/world-cups/{worldCupId}/available-rounds
```

Response: HTTP 200

```json
{
  "code": 1,
  "message": "플레이 가능한 라운드 조회 성공",
  "data": {
    "worldCupId": 1,
    "worldCupTitle": "제목",
    "worldCupDescription": "설명",
    "rounds": [2, 4, 8, 16]
  }
}
```

규칙:

- 지원 라운드 후보: `2`, `4`, `8`, `16`, `32`, `64`, `128`, `256`
- 공개·사용 가능한 콘텐츠 수 이하의 라운드만 반환
- 게임 조회 수를 이 API 호출 시 증가시키는 기존 규칙은 재검토한다. 단순 화면 진입과 실제 플레이 시작을 구분하는 것이 좋다.

오류:

- HTTP 404: 월드컵 없음
- HTTP 400: 플레이 가능한 콘텐츠 부족

### 5.3 게임 콘텐츠 조회

```http
GET /api/world-cups/{worldCupId}/contents
```

Query:

| 이름 | 타입 | 필수 | 규칙 |
| --- | --- | --- | --- |
| `currentRound` | integer | 예 | 지원 라운드 중 하나 |
| `sliceContents` | integer | 예 | 1~4 |
| `excludeContentsIds` | comma-separated integer | 아니오 | 이미 선택·탈락한 콘텐츠 ID |

Response: HTTP 200

```json
{
  "code": 1,
  "message": "컨텐츠 조회 성공",
  "data": {
    "worldCupId": 1,
    "title": "제목",
    "round": 16,
    "contentsList": [
      {
        "fileType": "STATIC_MEDIA_FILE",
        "contentsId": 10,
        "name": "후보 A",
        "mediaFileId": 100,
        "internetMovieStartPlayTime": null,
        "playDuration": null
      }
    ]
  }
}
```

오류:

- HTTP 400: 지원하지 않는 라운드
- HTTP 400: 요청 수량과 조회 수량 불일치
- HTTP 400: 제외한 콘텐츠가 다시 조회됨
- HTTP 404: 월드컵 없음

확인 필요:

- 프론트엔드 타입은 `playDuration`이 아니라 `videoPlayDuration`을 선언하고 있다.
- 한 건만 반환되는 상황에서 기존 Spring의 shuffle 반복문이 종료되지 않을 수 있으므로 NestJS에서는 안전하게 구현한다.

### 5.4 게임 종료와 결과 저장

```http
POST /api/world-cups/{worldCupId}/clear
Content-Type: application/json
```

현재 request:

```json
{
  "firstWinnerContentsId": 1,
  "secondWinnerContentsId": 2,
  "thirdWinnerContentsId": 3,
  "fourthWinnerContentsId": 4
}
```

Response: HTTP 201

```json
{
  "code": 1,
  "message": "게임 결과 생성",
  "data": [
    {
      "contentsName": "후보 A",
      "contentsId": 1,
      "mediaFileId": 10,
      "rank": 1
    }
  ]
}
```

기존 점수 규칙:

- 1위: +10
- 2위: +7
- 3위: +4
- 4위: +4

반드시 결정:

- 2강 게임에서는 3·4위 ID가 존재하지 않는다. 프론트엔드는 `0`을 `undefined`로 바꾸지만 Spring DTO는 primitive `int`이며 서비스는 항상 네 개의 ID를 요구한다.
- 신규 계약은 순위 배열 형태로 단순화하는 것을 우선 검토한다.
- 요청한 콘텐츠가 실제 해당 월드컵에 속하는지 검증한다.
- 동일 플레이 결과의 중복 제출 방지 정책을 정한다.

### 5.5 게임 결과와 랭킹

```http
GET /api/world-cups/{worldCupId}/game-result-contents
```

Response: HTTP 200

```json
{
  "code": 1,
  "message": "게임 결과 컨텐츠 리스트 조회 성공",
  "data": [
    {
      "contentsId": 1,
      "contentsName": "후보 A",
      "mediaFileId": 10,
      "gameRank": 1,
      "gameScore": 100
    }
  ]
}
```

현재 규칙은 콘텐츠 누적 `gameScore` 내림차순으로 순위를 계산한다. 동점 순위 처리와 안정적인 보조 정렬 기준은 새로 정의해야 한다.

## 6. 회원과 인증 API

### 6.1 회원가입

```http
POST /api/members/sign-up
Content-Type: application/json
```

Request:

```json
{
  "serviceId": "member01",
  "nickname": "동민",
  "password": "Example1!"
}
```

Response: HTTP 201

```json
{
  "code": 1,
  "message": "가입 성공",
  "data": null
}
```

오류:

- HTTP 400: validation 실패
- HTTP 409: service ID 또는 nickname 중복

검증 규칙 충돌:

| 필드 | 프론트엔드 | 기존 Spring | NestJS 결정 필요 |
| --- | --- | --- | --- |
| `serviceId` | 필수, 최대 10자 | 6~20자, 공백 금지 | 예 |
| `nickname` | 필수 | 2~10자, 공백 금지 | 예 |
| `password` | 8~16자, 영문·숫자·특수문자 | 6자 이상, 공백 금지 | 예 |

신규 기준을 먼저 정하고 프론트엔드와 NestJS validation을 동일하게 맞춘다.

### 6.2 로그인

```http
POST /api/members/sign-in
Content-Type: application/json
```

Request:

```json
{
  "serviceId": "member01",
  "password": "Example1!"
}
```

현재 response:

- HTTP 200
- body: `{ "code": 1, "message": "로그인 성공", "data": null }`
- header: `access-token`
- header: `refresh-token`

오류:

- HTTP 400: validation 실패
- HTTP 404, errorCode `5003`: 회원 또는 비밀번호 불일치

재설계 검토:

- 계정 존재 여부가 노출되지 않도록 로그인 실패를 동일한 HTTP 401 응답으로 통일
- 토큰을 응답 헤더로 유지할지 body 또는 HttpOnly cookie 방식으로 변경할지 결정

### 6.3 내 정보 요약

```http
GET /api/members/me/summary
access-token: <token>
```

Response: HTTP 200

```json
{
  "code": 1,
  "message": "정보 조회 성공",
  "data": {
    "memberId": 1,
    "serviceId": "member01",
    "nickname": "동민"
  }
}
```

프론트엔드 타입은 `memberId`가 아니라 `id`를 기대하고 있으므로 필드명을 반드시 통일한다.

### 6.4 로그아웃

```http
GET /api/members/sign-out
access-token: <token>
```

현재 response: HTTP 204

재설계 검토:

- 상태 변경 요청이므로 `POST /api/auth/sign-out` 형태를 우선 검토
- access token blacklist 대신 refresh token session/family 폐기를 기본으로 사용

### 6.5 토큰 갱신

```http
POST /api/new-access-token
Content-Type: application/json
```

현재 request:

```json
{
  "accessToken": "old-access-token",
  "refreshToken": "refresh-token"
}
```

현재 성공 response:

```json
{
  "code": 1,
  "message": "엑세스 토큰 생성",
  "data": {
    "newAccessToken": "new-access-token",
    "refreshToken": "refresh-token"
  }
}
```

현재 실패 response는 HTTP 200과 `code=1010101`을 사용한다.

신규 인증에서는 refresh token rotation, hash 저장, `jti`, `familyId`, 폐기와 재사용 감지를 설계한 뒤 이 API 계약을 다시 확정한다.

## 7. 월드컵 관리 API

이 절의 모든 API는 인증이 필수이며, 요청자가 해당 월드컵의 소유자인지 검사한다.

### 7.1 내 월드컵 목록

```http
GET /api/me/game-manage/world-cups
access-token: <token>
```

Response: HTTP 200

```json
{
  "code": 1,
  "message": "자신의 게임 리스트 조회",
  "data": [
    {
      "worldCupId": 1,
      "title": "제목",
      "description": "설명"
    }
  ]
}
```

프론트엔드 타입에는 `visibleType`도 선언되어 있으나 기존 Spring 응답 DTO에는 없다. 신규 응답에는 `visibleType`을 포함하는 방향을 권장한다.

### 7.2 내 월드컵 상세

```http
GET /api/me/game-manage/world-cups/{worldCupId}
access-token: <token>
```

Response: HTTP 200

```json
{
  "code": 1,
  "message": "자신의 월드컵 조회",
  "data": {
    "worldCupId": 1,
    "title": "제목",
    "description": "설명",
    "visibleType": "PUBLIC",
    "createdAt": "2026-01-01T00:00:00Z",
    "updatedAt": "2026-01-01T00:00:00Z"
  }
}
```

### 7.3 월드컵 생성

```http
POST /api/me/game-manage/world-cups
access-token: <token>
Content-Type: application/json
```

Request:

```json
{
  "title": "제목",
  "description": "설명",
  "visibleType": "PUBLIC"
}
```

규칙:

- `title`: 필수
- `description`: 선택, 최대 100자
- `visibleType`: `PUBLIC` 또는 `PRIVATE`
- 기존 Spring은 사용자별 또는 전체 title 중복 여부를 확인하므로 실제 범위를 확인해야 한다.

Response: HTTP 201

```json
{
  "code": 1,
  "message": "게임 생성",
  "data": 1
}
```

### 7.4 월드컵 삭제

```http
DELETE /api/me/game-manage/world-cups/{worldCupId}
access-token: <token>
```

Response: HTTP 204, body 없음

삭제 정책:

- 요청자 소유권 확인
- 관련 콘텐츠, 댓글, 게임 결과와 미디어의 처리 규칙 결정
- soft delete 또는 hard delete 여부 결정
- 오브젝트 스토리지 파일 삭제의 트랜잭션·재시도 정책 결정

## 8. 콘텐츠 관리 API

이 절의 모든 API는 인증과 월드컵 소유권 검사가 필수다.

공통 enum:

- `visibleType`: `PUBLIC`, `PRIVATE`
- `fileType`: `STATIC_MEDIA_FILE`, `INTERNET_VIDEO_URL`
- `detailFileType`: `GIF`, `PNG`, `JPEG`, `JPG`, `YOU_TUBE_URL`

### 8.1 관리용 콘텐츠 목록

```http
GET /api/me/game-contents-manage/world-cups/{worldCupId}/manage-contents
access-token: <token>
```

Response: HTTP 200

```json
{
  "code": 1,
  "message": "자신의 게임 컨텐츠 리스트 조회",
  "data": [
    {
      "worldCupId": 1,
      "contentsName": "후보 A",
      "mediaFileId": 10,
      "visibleType": "PUBLIC",
      "gameRank": 1,
      "gameScore": 100
    }
  ]
}
```

확인 필요:

- 기존 `GetMyWorldCupContentsResponse`는 첫 필드 이름이 `worldCupId`지만 실제로 콘텐츠 ID를 넣는다.
- 프론트엔드도 이 값을 `contentsId`로 변환하므로 신규 API에서는 처음부터 `contentsId`로 반환한다.
- 프론트엔드 타입은 `fileType`, `mediaPath`, `detailFileType`, `originalName` 등을 기대하지만 기존 목록 응답에는 없다. 현재는 미디어 조회 API를 추가 호출해 결합한다.

### 8.2 콘텐츠 일괄 생성

```http
POST /api/me/game-contents-manage/world-cups/{worldCupId}/contents
access-token: <token>
Content-Type: application/json
```

Request:

```json
{
  "data": [
    {
      "contentsName": "후보 A",
      "visibleType": "PUBLIC",
      "createMediaFileRequest": {
        "fileType": "STATIC_MEDIA_FILE",
        "detailFileType": "PNG",
        "mediaData": "data:image/png;base64,...",
        "originalName": "candidate.png",
        "videoStartTime": null,
        "videoPlayDuration": null
      }
    }
  ]
}
```

Response: HTTP 201

현재 프론트엔드는 이미지를 base64 data URL로 JSON에 포함한다. 신규 미디어 저장 구조를 정할 때 multipart upload 또는 presigned URL로 바꿀지 결정해야 한다.

### 8.3 콘텐츠 수정

```http
PUT /api/me/game-contents-manage/world-cups/{worldCupId}/contents/{contentsId}
access-token: <token>
Content-Type: application/json
```

Request:

```json
{
  "contentsName": "후보 A",
  "originalName": "candidate.png",
  "mediaData": "data:image/png;base64,...",
  "detailFileType": "PNG",
  "videoStartTime": null,
  "videoPlayDuration": null,
  "visibleType": "PUBLIC"
}
```

Response: HTTP 204, body 없음

### 8.4 콘텐츠 삭제

```http
DELETE /api/me/game-contents-manage/world-cups/{worldCupId}/contents/{contentsId}
access-token: <token>
```

Response: HTTP 204, body 없음

최소 플레이 인원 이하로 콘텐츠가 줄어드는 경우 월드컵의 플레이 가능 상태 처리 규칙을 정한다.

## 9. 댓글 API

### 9.1 댓글 목록

```http
GET /api/world-cups/{worldCupId}/comments?offset=0
```

Response: HTTP 200

```json
{
  "code": 1,
  "message": "코멘트 조회 성공",
  "data": [
    {
      "commentId": 1,
      "commentWriterId": 1,
      "writerNickname": "동민",
      "body": "댓글",
      "createdAt": "2026-01-01T00:00:00Z"
    }
  ]
}
```

현재는 offset만 존재한다. 신규 API에서는 `limit`과 안정적인 정렬 기준을 추가하되 프론트엔드 변경과 함께 진행한다.

### 9.2 댓글 작성

```http
POST /api/world-cups/{worldCupId}/contents/{contentsId}/comments
Content-Type: application/json
access-token: <optional-token>
```

Request:

```json
{
  "body": "댓글",
  "nickname": "비회원 닉네임"
}
```

Response: HTTP 201

```json
{
  "code": 1,
  "message": "댓글 작성",
  "data": null
}
```

규칙:

- `body`: 1~30자
- 로그인 사용자는 token의 회원 ID를 작성자로 사용
- 비회원 댓글을 계속 허용할지, 비회원 nickname 중복과 삭제 권한을 어떻게 처리할지 결정

## 10. 미디어 API

### 10.1 미디어 조회

```http
GET /api/media-files/{mediaFileId}?size=original
```

Query:

- `size`: `original` 또는 `divide2`, 기본 `original`

Response: HTTP 200

```json
{
  "code": 1,
  "message": "미디어 파일 조회",
  "data": {
    "mediaFileId": 1,
    "fileType": "STATIC_MEDIA_FILE",
    "mediaData": "data:image/png;base64,...",
    "originalName": "candidate.png",
    "videoStartTime": null,
    "videoPlayDuration": null,
    "detailType": "PNG",
    "createdAt": "2026-01-01T00:00:00Z",
    "updatedAt": "2026-01-01T00:00:00Z"
  }
}
```

신규 구조 결정:

- PostgreSQL에는 object key와 metadata만 저장한다.
- 실제 파일은 오브젝트 스토리지에 저장한다.
- API가 base64 전체를 반환할지, 서명되거나 공개된 URL을 반환할지 결정한다.
- 캐시 정책과 썸네일 생성 방식을 결정한다.

## 11. 확인된 불일치와 결정 목록

구현 전에 반드시 결정해야 하는 항목:

1. 인증 헤더를 기존 `access-token`으로 유지할지 표준 Bearer token으로 변경할지
2. refresh token을 body, response header, HttpOnly cookie 중 어디로 전달할지
3. 로그인 실패를 기존 404/5003으로 유지할지 401로 통일할지
4. `GET /members/me/summary`의 회원 ID 필드를 `memberId` 또는 `id` 중 무엇으로 통일할지
5. 회원가입의 ID, nickname, password validation 규칙
6. 게임 플레이 응답의 `playDuration`과 `videoPlayDuration` 필드명
7. 2강 게임 종료 요청에서 3·4위가 없는 경우의 표현
8. 월드컵 목록의 정렬 allowlist와 동점 보조 정렬
9. 랭킹을 누적 score로 계산할지 게임 결과 이벤트에서 집계할지
10. 비회원 댓글 허용과 삭제 권한
11. 이미지 전달을 base64로 유지할지 multipart 또는 presigned upload로 바꿀지
12. 월드컵·콘텐츠·댓글·미디어의 soft/hard delete 정책

## 12. NestJS 구현 우선순위

### 1차: 공개 읽기

1. `GET /api/world-cups`
2. `GET /api/world-cups/{worldCupId}/available-rounds`
3. `GET /api/world-cups/{worldCupId}/contents`
4. `GET /api/media-files/{mediaFileId}`
5. `GET /api/world-cups/{worldCupId}/game-result-contents`

### 2차: 게임 결과와 댓글

1. `POST /api/world-cups/{worldCupId}/clear`
2. `GET /api/world-cups/{worldCupId}/comments`
3. `POST /api/world-cups/{worldCupId}/contents/{contentsId}/comments`

### 3차: 회원과 인증

1. 회원가입과 로그인
2. 내 정보
3. refresh token rotation
4. 로그아웃과 session 폐기

### 4차: 관리 기능

1. 월드컵 생성·조회·삭제
2. 콘텐츠 목록·생성·수정·삭제
3. 미디어 업로드
4. 월드컵 수정과 댓글 삭제 등 현재 프론트엔드 미사용 기능

## 13. 다음 작업

1. 이 문서의 결정 목록을 하나씩 확정한다.
2. 확정된 계약으로 OpenAPI 초안을 작성한다.
3. PostgreSQL의 최소 테이블과 관계를 설계한다.
4. 신규 NestJS 저장소에서 공개 읽기 API부터 구현한다.

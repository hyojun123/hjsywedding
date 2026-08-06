# 모바일 청첩장

단일 페이지(`index.html`) 정적 사이트입니다. 빌드 도구·의존성 없이 파일 하나로 동작합니다.

- 배포: GitHub Pages (`master` 브랜치 루트)
- 이전 Jekyll 블로그는 `backup-blog` 브랜치에 그대로 보존되어 있습니다.

구성: 커버 → 인사말 → 신랑·신부 → 예식 일시(캘린더·카운트다운) → 갤러리 →
오시는 길(카카오맵) → 마음 전하실 곳 → **참석 여부(RSVP)** → 방명록 → 공유하기

## 수정하는 곳

### 1) 텍스트 — `index.html` 본문
`<!-- ✏️ -->` 주석이 달린 곳만 고치면 됩니다.

| 항목 | 위치 |
|---|---|
| 이름 / 예식일 / 예식장 | 커버 섹션 |
| 인사말, 혼주 성함 | 인사말 섹션 |
| 신랑·신부 소개, 전화·문자 번호 (`tel:`, `sms:`) | 신랑·신부 섹션 |
| 지하철·버스·주차 안내 | 오시는 길 섹션 |
| 참석 여부 안내 문구 | R.S.V.P 섹션 |
| 카톡 공유 미리보기 문구 | `<head>`의 `<title>`, `og:` 태그 |

### 2) 설정값 — `index.html` 하단 `CONFIG`
예식 일시(캘린더·D-day 자동 계산), 예식장 좌표, 갤러리 사진 목록, 계좌번호, API 키.

### 3) 사진 — `images/`
`images/README.md` 참고. 넣기 전에는 자리표시자가 표시됩니다.

## 예식장 좌표 찾기

1. https://map.kakao.com 에서 예식장 검색
2. 지도에서 우클릭 → **좌표 복사** (또는 URL의 좌표값 확인)
3. `CONFIG.venue.lat` / `lng`에 입력

## 카카오 설정

`CONFIG.kakaoJsKey`에 **JavaScript 키**가 들어 있습니다. (JS 키는 클라이언트에 노출되는 것이 정상입니다. **REST API 키는 절대 이 저장소에 넣지 마세요.**)

지도와 카톡 공유가 동작하려면 **Web 플랫폼에 도메인을 등록**해야 합니다. 이것 하나면 됩니다.
(지도 SDK에는 "카카오맵 활성화" 같은 별도 토글이 없습니다 — 찾지 마세요.)

1. [developers.kakao.com](https://developers.kakao.com) 로그인 → 상단 **내 애플리케이션**
2. JS 키가 `8cb17b61...`인 앱 선택
   (앱이 여러 개면 왼쪽 **앱 설정 → 앱 키**에서 키가 일치하는지 확인. 다른 앱에 등록하면 계속 401)
3. 왼쪽 **앱 설정 → 플랫폼** → 페이지 하단 **Web** 영역의 `Web 플랫폼 등록`
4. **사이트 도메인**에 아래를 입력하고 저장

```
https://hyojun123.github.io
http://localhost:8000
```

> 등록 여부는 이 명령으로 확인할 수 있습니다 (`200`이면 정상, `401`이면 미등록):
> ```bash
> curl -s -o /dev/null -w "%{http_code}\n" -H "Referer: https://hyojun123.github.io/" \
>   "https://dapi.kakao.com/v2/maps/sdk.js?appkey=8cb17b61b0c4374685d01eeb5c845d3c"
> ```

## 데이터 저장소 (방명록 · 참석 여부)

`CONFIG.firebase`가 비어 있으면 **작성한 브라우저에만 저장되는 미리보기 모드**로 동작합니다.
하객들의 응답을 실제로 받으려면 Firebase 연결이 필요합니다.

### 어떤 걸 써야 하나 — Firestore(NoSQL) vs Data Connect(SQL)

| | **Cloud Firestore** (NoSQL) | **Data Connect** (SQL) |
|---|---|---|
| 저장 형태 | 컬렉션 / 문서(JSON) | PostgreSQL 테이블 |
| 스키마 | 없음 (앱이 쓰는 필드가 곧 스키마) | GraphQL SDL로 미리 정의 |
| 요금제 | **Spark(무료)** 가능 | **Blaze(종량제) 필수** |
| 실비용 | 청첩장 규모면 사실상 0원 | Cloud SQL 인스턴스가 상시 과금 (월 1만원~) |
| 배포 | 웹 SDK만 넣으면 끝 | `firebase deploy`로 스키마 배포 필요 |

**청첩장에는 Firestore를 쓰세요.** 데이터가 방명록 몇백 건 수준이고 조인·집계가 없어서
SQL의 장점이 나올 구석이 없는데, Data Connect는 Cloud SQL 인스턴스가 켜져 있는 내내 돈이 나갑니다.
아래 설정은 모두 Firestore 기준입니다.

### 설정 순서

1. [Firebase 콘솔](https://console.firebase.google.com)에서 프로젝트 생성
2. **빌드 → Firestore Database** 만들기 (프로덕션 모드, 위치 `asia-northeast3`)
3. **프로젝트 설정 → 내 앱 → 웹 앱 추가** 후 나오는 값을 `CONFIG.firebase`에 붙여넣기
   (`apiKey`, `authDomain`, `projectId`, `appId`)
4. **Firestore → 규칙**을 아래 규칙으로 교체 후 게시

> `apiKey`는 비밀값이 아니라 프로젝트 식별자입니다. 클라이언트에 노출되는 게 정상이고,
> 실제 접근 통제는 아래 **보안 규칙**이 담당합니다. 규칙을 열어두면 그대로 뚫립니다.

### 스키마

Firestore는 스키마가 없는 DB라, **아래 표가 곧 이 페이지가 읽고 쓰는 필드 규약**입니다.
컬렉션·문서는 첫 저장 시 자동 생성되므로 콘솔에서 미리 만들 필요 없습니다.
문서 ID는 자동 생성이고, 정렬은 `at` 필드(내림차순)로 합니다. 단일 필드 인덱스는 자동이라 별도 인덱스 설정이 없습니다.

**`guestbook` 컬렉션** — 축하 메시지 (공개)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `name` | string | ✓ | 작성자 이름, 최대 10자 |
| `msg` | string | ✓ | 메시지 본문, 최대 300자 |
| `pw` | string | ✓ | 삭제용 비밀번호 4자리의 SHA-256 해시 (64자 hex) |
| `at` | number | ✓ | 작성 시각, epoch milliseconds — 정렬 키 |

**`rsvp` 컬렉션** — 참석 여부 (비공개, 콘솔에서만 조회)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `side` | string | ✓ | `'신랑측'` \| `'신부측'` |
| `attend` | string | ✓ | `'참석'` \| `'불참'` |
| `name` | string | ✓ | 성함, 최대 10자 |
| `count` | number | ✓ | 참석 인원(본인 포함) 1~20. 불참이면 `0` |
| `meal` | string | ✓ | `'식사함'` \| `'식사안함'`. 불참이면 빈 문자열 |
| `phone` | string | ✓ | 연락처(선택 입력) — 미입력 시 빈 문자열 |
| `memo` | string | ✓ | 전하실 말씀(선택), 최대 200자 — 미입력 시 빈 문자열 |
| `at` | number | ✓ | 제출 시각, epoch milliseconds |

> 선택 항목도 항상 빈 문자열로 채워 보냅니다. 보안 규칙의 `hasOnly` 검사와 필드 존재 여부를
> 단순하게 유지하기 위해서입니다.

집계된 참석자 명단은 **Firebase 콘솔 → Firestore → `rsvp` 컬렉션**에서 확인하세요.
페이지에서는 `rsvp`를 절대 읽지 않습니다 (하객 명단·연락처가 공개되면 안 되므로).

### 보안 규칙

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // 방명록: 누구나 읽고 쓸 수 있음
    match /guestbook/{doc} {
      allow read: if true;
      allow create: if request.resource.data.keys().hasOnly(['name','msg','pw','at'])
                    && request.resource.data.name is string
                    && request.resource.data.name.size() > 0
                    && request.resource.data.name.size() <= 10
                    && request.resource.data.msg is string
                    && request.resource.data.msg.size() > 0
                    && request.resource.data.msg.size() <= 300
                    && request.resource.data.pw is string
                    && request.resource.data.at is number;
      allow delete: if true;   // 비밀번호 대조는 브라우저에서 수행 (정적 사이트 한계)
      allow update: if false;
    }

    // 참석 여부: 제출만 가능, 조회는 콘솔에서만
    match /rsvp/{doc} {
      allow read, update, delete: if false;
      allow create: if request.resource.data.keys().hasOnly(['side','attend','name','count','meal','phone','memo','at'])
                    && request.resource.data.side in ['신랑측','신부측']
                    && request.resource.data.attend in ['참석','불참']
                    && request.resource.data.name is string
                    && request.resource.data.name.size() > 0
                    && request.resource.data.name.size() <= 10
                    && request.resource.data.count is number
                    && request.resource.data.count >= 0
                    && request.resource.data.count <= 20
                    && request.resource.data.memo.size() <= 200
                    && request.resource.data.at is number;
    }
  }
}
```

> **알아두실 점**
> - 방명록 비밀번호는 SHA-256으로 해시해 저장하고, 삭제 시 브라우저에서 대조합니다.
>   서버가 없으므로 규칙상 삭제 자체는 막을 수 없습니다. 예식이 끝나면
>   `allow delete: if true` → `if false`로 바꿔 잠그는 것을 권합니다.
> - 참석 여부는 `allow read: if false`라 페이지에서 조회되지 않지만, 누구나 제출은 가능합니다.
>   접수를 마감하려면 `allow create`를 `if false`로 바꾸세요.

## 로컬에서 확인

```bash
python3 -m http.server 8000
# http://localhost:8000
```

브라우저 개발자도구에서 모바일(iPhone 등) 화면으로 보면 실제와 가장 비슷합니다.

## 배포

`master` 브랜치에 푸시하면 1~2분 뒤 https://hyojun123.github.io 에 반영됩니다.
(저장소 Settings → Pages → Source: `Deploy from a branch` / `master` / `/ (root)`)

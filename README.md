# 기업마당 지원사업 공고 대시보드

기업마당(bizinfo.go.kr) 지원사업 전체공고를 한 화면에 모으고, Cloudflare Worker로
**매일 자동 갱신**하는 단일 HTML 대시보드.

```
index.html   대시보드 본체 (단일 파일, 의존성 없음, 내장 스냅샷 포함)
worker.js    Cloudflare Workers — 공고 수집 API
collect.py   초기 스냅샷 수집 (목록 + 상세)
files.py     첨부파일 링크 수집
build.py     raw.json → notices.json 정규화
README.md    이 문서
```

## 1. 동작 구조

```
[브라우저 index.html]
   │ ① 내장 스냅샷(BASE)으로 즉시 렌더 — 워커 없이도 동작
   │ ② 열 때 마지막 갱신 후 24시간 경과 → fetch /bizinfo/list
   │                                          │ 엑셀 다운로드 엔드포인트 1회 요청
   │                                          ▼  (xlsx = ZIP → sheet1.xml 파싱)
   │                                     bizinfo.go.kr
   │ ③ 전건 교체 + 신규 공고 NEW 배지 + localStorage 저장
   │ ④ 모달 열 때 fetch /bizinfo/detail?id=... (사업개요·첨부·태그)
```

관심공고(★)·메모는 공고 ID 기준으로 따로 보관하므로 갱신해도 유지된다.

## 2. 수집 경로 — 왜 엑셀인가

| 경로 | 결과 |
|---|---|
| 공개 OpenAPI `/uss/rss/bizinfoApi.do` | **인증키 필요** → 사용 불가 |
| 목록 페이지 `selectSIIA200View.do?rows=100` | rows/cpage 무시, 항상 15건 |
| 같은 경로 POST | 403 (CSRF 토큰) |
| **`selectSIIA200ExcelDownload.do`** | **인증 없이 전건(1,577행) 반환** ✅ |

```
GET /sii/siia/selectSIIA200ExcelDownload.do?1=1&schEndAt=N    # 진행중
GET /sii/siia/selectSIIA200ExcelDownload.do?1=1&schEndAt=Y    # 지난 공고
```
컬럼: 번호 / 소관부처 / 사업수행기관 / 지원분야 / 공고명 / 신청시작일자 / 신청종료일자 / 등록일자 / 공고상세URL

이 xlsx는 **스트리밍 ZIP**(로컬 헤더의 크기 필드가 0, data descriptor 사용)이라
중앙 디렉터리에서 오프셋·압축크기를 읽어야 한다. 워커는 `DecompressionStream('deflate-raw')`로
풀고 `sheet1.xml`의 `<row>/<c>`를 정규식으로 읽는다(1,577행 파싱 ~10ms).

### 상세 페이지
```
GET /sii/siia/selectSIIA200Detail.do?pblancId=PBLN_000000000126545
```
- `<span class="s_title">라벨</span><div class="txt">` 패턴.
  존재하는 라벨은 **소관부처·지자체 / 사업수행기관 / 신청기간 / 사업개요 / 사업신청 방법 /
  사업신청 사이트 / 문의처** 뿐이다(지원대상·지원내용 라벨은 없음).
- 첨부파일: `<div class="attached_file_list">` 내부의 `file_name` +
  `/cmm/fms/fileDown.do?atchFileId=FILE_...&fileSn=N` (인증 없이 200).
  ⚠ `fileBlank()`의 `/webapp/upload/...` 경로는 **404** — 쓰지 말 것.
- 해시태그: `<span style="color:#4098a8">#태그</span>`.
  ⚠ **전국 17개 시도가 관행적으로 전부 붙는다** → 지역 판정 근거로 쓰면 안 된다.
  지역은 소관부처(지자체명) + 공고명 앞 `[지역]` 표기로만 판정(전국사업 379건은 지역 공백).

## 3. 워커 API

| 경로 | 설명 |
|---|---|
| `/bizinfo/list?stt=open&limit=3000` | 진행중 공고 전건. `stt=end`는 지난 공고. 캐시 1시간 |
| `/bizinfo/detail?id=PBLN_...` | 사업개요·신청방법·문의처·신청사이트·해시태그·첨부. 캐시 24시간 |
| `/bizinfo/detail?id=A,B,C` | 최대 12건 동시 |
| `/?url=<주소>` | CORS 중계 (bizinfo.go.kr 만 허용) |

목록 응답:
```json
{ "updatedAt":"2026-09-16T07:00:00.000Z", "count":1577,
  "items":[{ "id":"PBLN_...","title":"...","field":"수출","dept":"중소벤처기업부",
             "org":"중소기업중앙회","start":"2026-09-16","end":"2026-10-07",
             "reg":"2026-09-16","gov":"중앙부처","regions":["경기"],"url":"https://..." }] }
```

## 4. 배포

1. dash.cloudflare.com → Workers & Pages → Create → Worker
2. `worker.js` 전체를 붙여넣고 Deploy
3. 발급 주소(예: `https://bizinfo-feed.내계정.workers.dev`)를 대시보드 **⚙ 자동갱신**에 입력
   → "연결 확인"으로 응답 점검 후 "저장 후 갱신"

R&D 대시보드 워커(`soft-glade-215e`)와는 별개 워커로 두는 것을 권한다
(기업마당은 목록 1회 요청이 1MB급이고 캐시 주기도 달라서, 한 워커에 합치면 서로의 캐시를 밀어낸다).

## 5. localStorage 키

| 키 | 내용 |
|---|---|
| `bizinfo-data` | 마지막 갱신 목록 (`{collected, items}`) — 다음 방문 시 내장 스냅샷 대신 사용 |
| `bizinfo-detail` | 워커에서 받은 상세 캐시 |
| `bizinfo-cfg` | `{worker, auto, lastSync}` |
| `bizinfo-new` | 마지막 갱신에서 새로 올라온 공고 ID (NEW 배지) |
| `bizinfo-fav` / `bizinfo-memo` | 관심공고 / 공고별 메모 (갱신과 무관하게 유지) |
| `bizinfo-state` / `bizinfo-theme` | 필터·정렬 상태 / 테마 |

## 6. 진단

| 증상 | 원인 / 확인 |
|---|---|
| "갱신 실패: HTTP 502" | 기업마당 엑셀 엔드포인트 응답 지연. 워커 로그의 `detail` 확인, 잠시 후 재시도 |
| "갱신 실패: 수신 0건" | 엑셀 컬럼 순서가 바뀐 경우. `listNotices()`의 인덱스(r[1]~r[8]) 점검 |
| 갱신은 되는데 저장이 안 됨 | localStorage 5MB 한계. 갱신 목록은 약 0.5MB라 보통 문제없음. 초과 시 그 세션만 메모리로 동작 |
| 상세가 계속 "불러오는 중" | 워커 주소 오타 또는 `/bizinfo/detail` 502. 브라우저 콘솔 확인 |
| D-day가 하루 밀림 | 정상. D-day는 열 때마다 브라우저 날짜로 재계산되고, 목록 자체는 24시간마다 갱신 |
| 지역 필터에 안 걸리는 공고 | 중앙부처 전국사업(379건)은 지역값이 없다. 지역 대신 `공고주체=중앙부처`로 보라 |

## 7. 개발 경과

- **2026-09-16 v1** — 1,577건 상세 전량 수집(파이썬), 단일 HTML 대시보드. 필터 5줄·검색·D-day·관심·메모·CSV.
- **2026-09-16 v2** — Cloudflare Worker 추가. 내장 스냅샷 + 하루 1회 자동 갱신, 신규 공고 NEW 배지,
  상세 지연 로드, D-day 브라우저 날짜 기준 재계산.
  (워커 목록에는 상세 필드가 없어 검색이 깨졌던 문제 → `norm()`으로 빈 값 보정 + DETAIL 캐시까지 검색 대상에 포함)

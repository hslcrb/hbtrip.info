# 시스템 아키텍처

## 개요

HBTrip.info는 정적 호스팅 환경(GitHub Pages) 위에서 동작하는 순수 HTML/CSS/JS 웹사이트입니다.
본 문서는 `newpages/`에 구현된 대화형 기능들의 설계를 설명합니다.

---

## 1. Git Museum (`newpages/museum.html`)

GitHub REST API로 커밋 히스토리를 실시간 조회하고, 각 커밋 시점의 파일을 프리뷰할 수 있는 Git 히스토리 브라우저입니다.

### 데이터 흐름

```
museum.html 로드
  └─ GET /repos/hslcrb/hbtrip.info/commits?per_page=100
       └─ sessionStorage('museumList') 캐시 (세션 유지)
       └─ 커밋 목록 렌더링 + 활동 차트

사용자가 커밋 행 클릭
  └─ GET /repos/hslcrb/hbtrip.info/commits/{sha}
       └─ sessionStorage('detail_{sha}') 캐시
       └─ 해당 커밋의 파일 목록 표시 + blob 링크

사용자가 'Preview' 버튼 클릭
  └─ GET raw.githubusercontent.com/{sha}/{path}
  └─ HTML 파싱 → <a href> 링크 재작성
  └─ Blob URL 생성 → iframe 렌더링 (오버레이)
  └─ location.hash = '#{path} @ {sha7}'
```

### 링크 재작성 엔진

프리뷰 내부의 `<a href>`를 자동으로 해당 파일이 존재하는 가장 가까운 이전 커밋으로 연결합니다.

```
[Flow]
raw HTML 내 모든 <a href="..."> 추출
  └─ resolvePath(): 상대경로 → repo 절대경로
  └─ findFileInHistory(): 현재 커밋부터 거꾸로 스캔
       └─ fetchDetail()로 각 커밋의 파일 목록 확인
       └─ 최초 발견 시 해당 sha7 반환
  └─ href="museum.html#{path} @ {sha}" 로 재작성
  └─ target="_parent" 추가 → iframe 밖에서 열림
```

외부 링크(http://, https://, #, mailto:, tel:, javascript:)는 skip됩니다.

### hash 기반 네비게이션

```
museum.html               → 커밋 목록 (기본)
museum.html#time.html @ a1b2c3d  → 파일 프리뷰 오버레이 열림
뒤로가기                  → hash 제거 → 오버레이 닫힘
```

`hashchange` 이벤트로 감지하며, X·Esc·배경 클릭 시 `history.back()` 호출.

### 커밋 타입 분류

| 접두사 | 타입 | 그래프 색상 |
|--------|------|------------|
| Add files via upload | upload | 초록 #3fb950 |
| Rename | rename | 황금 #d29922 |
| Update / 업데이트 | update | 파랑 #58a6ff |
| Create / 생성 | create | 보라 #bc8cff |
| 그 외 | misc | 회색 #8b949e |

### 활동 차트

월별 커밋 수를 막대 그래프로 표시. 높이는 최대값 기준 정규화.

---

## 2. 타임라인 시스템 (`newpages/timeline.html`)

VERSIONS 배열에 정의된 각 버전을 슬라이드와 카드로 탐색하는 인터페이스입니다.

### 버전 데이터

```js
const VERSIONS = [
  { id: 'vfinal', label: 'Final', file: 'index(last).html', ver: 'Final', date: '2025-08-20' },
  ...
];
```

Git 로그에서 추출한 날짜로 정렬. `id` 값은 URL hash로 사용됩니다.

### 네비게이션

- 좌우 화살표 / 키보드 ← →
- URL hash: `timeline.html#index(last)` → 페이지 로드 시 해당 버전으로 이동
- 각 버전 카드에는 페이지 스크린샷, 버전 라벨, 날짜, 파일명 뱃지 표시

### 연결된 페이지

- 헤더 우측 'Git 뮤지엄' 버튼 → `museum.html`
- 푸터 'Git 뮤지엄' 링크
- Standalone 섹션: `home.html`, `time.html`, `ready.html`

---

## 3. Time Settings (`newpages/time.html`)

카운트다운 페이지 상단에 고정된 시간 설정 바.

### 세 가지 모드

| 모드 | 동작 |
|------|------|
| 현재시간 | 서버 RTT 보정 시간 실시간 표시 |
| 직접설정 | 사용자가 직접 날짜 선택 |
| 타임라인 | VERSIONS 배열 기준 버전 선택 → 해당 날짜 적용 |

### 상태 관리

```js
let timeMode = 'realtime';     // 'realtime' | 'custom' | 'timeline'
let customDate = null;         // Date 객체
let timelineVersionIdx = -1;   // VERSIONS 인덱스
```

쿠키(`timeStartup`, 24시간) 저장 → 재방문 시 자동 복원.

### RTT 보정

```
fetch('/time.html', { method: 'HEAD' })
  └─ 왕복 시간 측정 → 편도 추정
  └─ 서버 Date 헤더 + RTT/2 → 보정된 현재 시간
```

---

## 4. 버전 네비게이션 (모든 버전 파일)

24개 버전 HTML 파일에 주입된 floating 타임라인 버튼.

```html
<div style="position:fixed;bottom:20px;right:70px;z-index:9999">
  <a href="newpages/timeline.html#{versionId}"
     style="...">⌛ 타임라인</a>
</div>
```

각 파일의 `{versionId}`는 VERSIONS 배열의 `id`와 일치.

---

## 5. 루트 Index (`index.html`)

- `<meta http-equiv="refresh">` → `newpages/timeline.html` (5초 후 자동 이동)
- 상단 파란색 띠 (#0737f7) + 메가폰 SVG → `newpages/timeline.html` 링크
- 기존 팝업을 제거하고 인페이지 배너로 대체

---

## 캐싱 전략

| 데이터 | 저장소 | 유효기간 |
|--------|--------|---------|
| 커밋 목록 | sessionStorage | 세션 |
| 커밋 상세 (파일 목록) | sessionStorage | 세션 |
| time 설정 | cookie | 24시간 |

GitHub API 인증 없음 → 60회/시간 rate limit. sessionStorage 캐싱으로 최소화.

---

## 파일 구조

```
/
├── index.html              # → newpages/timeline.html 리다이렉트
├── newpages/
│   ├── timeline.html       # 버전 타임라인 (진입점)
│   ├── museum.html         # Git 커밋 브라우저
│   ├── time.html           # 카운트다운 + 시간 설정
│   ├── home.html           # 학과 포털
│   ├── ready.html          # 준비 페이지
│   ├── sigak.html          # 시각디자인 일정
│   ├── char.html           # 뷰티/캐릭터 일정
│   ├── bdsc.html           # 빅데이터/스마트제어 일정
│   ├── sigak(22-2edit).html  # 버전 파일 (24개)
│   └── ...
├── docs/
│   ├── architecture.md     # 본 문서
│   └── version-history.md  # 변경 이력
└── README.md
```

---

*2025-2026 Rheehose (Rhee Creative)*

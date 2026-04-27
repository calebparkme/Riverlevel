# CLAUDE.md

이 파일은 Claude Code (claude.ai/code)가 이 저장소에서 작업할 때 참고하는 가이드입니다.

## 프로젝트 개요

온타리오 강 수위를 실시간으로 모니터링하는 단일 파일 정적 웹 대시보드(`index.html`)입니다. 빌드 시스템이나 설치할 의존성이 없으며, 브라우저에서 파일을 직접 열거나 정적 파일 서버로 제공하면 됩니다.

## 로컬 실행

```bash
# 아래 중 아무 방법이나 사용 가능:
open index.html
python3 -m http.server 8080
npx serve .
```

## 아키텍처

전체 애플리케이션이 `index.html` 하나에 담겨 있습니다 — `<style>`에 CSS, `<body>`에 마크업, `<script>` 블록 하나에 모든 로직이 들어 있습니다. 프레임워크, 번들러, 서버 사이드 코드가 없습니다.

**CDN 외부 의존성:**
- Chart.js 4.4.0 (`chart.umd.min.js`)
- chartjs-adapter-date-fns 3.0.0 (시간 축 파싱용)

**데이터 소스 (모두 CORS 허용, 프록시 불필요):**
- 관측소 목록: `https://api.weather.gc.ca/collections/hydrometric-stations/items?PROV_TERR_STATE_LOC=ON&STATUS_EN=Active`
- 실시간 수위: `https://api.weather.gc.ca/collections/hydrometric-realtime/items?STATION_NUMBER=...`

## 핵심 아키텍처 패턴

**초기화 순서** (`init()`):
1. `loadAllStations()` — 온타리오 전체 활성 관측소를 가져와 `ALL_STATIONS`(REAL_TIME=1)와 `EXCLUDED_STATIONS`로 분류
2. `renderNotePanel()` — 실시간 데이터가 없는 관측소 목록을 사이드바에 렌더링
3. `loadSettings()` — `localStorage`에서 패널 레이아웃 및 설정 복원
4. `buildPanels()` + `initDragAndDrop()` — 9개 패널의 DOM 생성
5. `refreshAll()` + `startAutoRefresh()` — 수위 데이터 로드 및 자동 갱신 타이머 시작

**세대 ID를 이용한 순차 로딩:**
`refreshAll()`은 API 속도 제한을 피하기 위해 패널을 병렬이 아닌 순차적으로 하나씩 로드합니다. 새로고침마다 `refreshId`가 증가하며, `loadPanel()`은 `expectedRefreshId`를 받아 더 새로운 새로고침이 시작됐으면 조용히 중단합니다 — 오래된 데이터가 최신 로드를 덮어쓰는 것을 방지합니다.

**모듈 레벨 상태 변수:**
- `ALL_STATIONS` / `EXCLUDED_STATIONS` — API에서 가져온 전체 관측소 목록
- `panelStations[]` — 현재 표시 중인 9개 관측소 (인덱스 → 관측소 객체)
- `stationData{}` — 캐시: `stationId → { level, discharge, time, history[] }`
- `chartInstances{}` — `panelIndex → Chart` 인스턴스; 재생성 전 반드시 `destroy()` 호출
- `fishingLevels{}` — 패널별 사용자 정의 "낚시 적정 수위" 기준선
- `currentRange` — `'Today' | '3 Days' | '1 Week' | '1 Month'` 중 하나
- `refreshIntervalMinutes` — 자동 갱신 주기 (10 / 30 / 60분)

**localStorage 영속성** (키: `grand-river-dashboard-v1`):
`currentRange`, `refreshIntervalMinutes`, `panelStationIds`, `fishingLevels`를 저장합니다. `loadSettings()`에는 이전 한글 범위 키에 대한 하위 호환 별칭이 포함되어 있습니다.

**차트 렌더링** (`drawStationChart()`):
각 패널은 Chart.js 시계열 선형 차트를 렌더링합니다. 커스텀 `thresholds` 플러그인이 `warnLevel`, `dangerLevel`, 사용자의 `fishingLevel`에 점선 수평선을 그립니다. 스크립트 상단의 `THRESHOLDS` 객체에 관측소 ID별 알려진 임계값이 정의되어 있습니다.

**패널 관측소 선택:**
각 패널에는 검색 가능한 드롭다운(`filterDropdown()`)이 있으며, 관측소 이름에서 추출한 강 이름(`extractRiverGroup()`)으로 그룹화됩니다. 관측소를 선택하면 `selectPanelStation()`이 호출되어 `panelStations[]`를 업데이트하고 설정을 저장한 뒤 해당 패널만 재로드합니다.

**웹캠 패널:**
정적 이미지 URL(30초마다 자동 갱신)과 YouTube 라이브 스트림(`youtube:` 접두사)을 지원합니다. 현재는 Grand River 카메라만 설정되어 있습니다.

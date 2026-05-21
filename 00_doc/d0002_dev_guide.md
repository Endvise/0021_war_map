# d0002 개발자 가이드 — SLG 전장 타워 시뮬레이터

> 최초 작성: 2026-05-21
> 다른 PC에서 코드 수정 시 이 문서를 먼저 읽을 것

---

## 1. 빠른 시작

```bash
# 레포 클론
git clone https://github.com/Endvise/0021_war_map.git
cd 0021_war_map

# 끝. 빌드 없음. 브라우저로 바로 열기
# index.html 파일을 브라우저에 드래그앤드롭 또는:
start index.html        # Windows
open index.html         # Mac
```

**배포 URL:** https://endvise.github.io/0021_war_map/

---

## 2. 프로젝트 구조

```
0021_war_map/
├── index.html              ← 전체 앱 (단일 파일, ~2700줄)
├── 00_doc/
│   ├── d0001_prd.md        ← 요구사항 문서
│   ├── d0002_dev_guide.md  ← 이 파일
│   └── d0010_history.md    ← 결정 기록
├── 아일랜드 좌표계산기.xlsx  ← 맵 좌표 계산용 스프레드시트
└── netlify.toml             ← Netlify 배포 설정 (현재 미사용)
```

**기술 스택:** 순수 HTML + CSS + Vanilla JS (빌드 없음, npm 없음)

---

## 3. index.html 섹션 지도

| 줄 번호 | 섹션 | 설명 |
|---------|------|------|
| 1~276 | `<head>` | CSS 스타일, Firebase CDN 스크립트 |
| 277~528 | `<body>` HTML | 헤더, 캔버스, 사이드패널 UI |
| 529~532 | 상수 | MAP_W, MAP_H, ppc(픽셀/칸=3) |
| 533~543 | 타워 비용 룩업테이블 | `TOWER_COST[]` — 실제 게임 데이터 |
| 544~652 | 시멘트 생산량 테이블 | `CEMENT_PROD[]` — 점유 도시 기반 |
| 653~706 | 연맹 상태 | `alliances[]` — 8개 연맹 데이터 |
| 707~727 | 점수 상수 | 도시 등급별 점수 |
| 728~750 | 도시 상태 | `cities[]` — 6/7/8도시 좌표·점유 |
| 751~822 | 캔버스·좌표 | `toC()`, `fromC()` 좌표 변환 |
| 820~845 | 드래그 | 타워/도시 드래그 상태 |
| 846~869 | 동맹 관계 | `relations{}` 매트릭스 |
| 870~1080 | 그리기 | `drawAll()` — 맵 전체 렌더링 |
| 1081~1135 | 경유지 깃발 | `waypoints[]`, `drawWaypoints()` |
| 1136~1155 | 충돌 마커 | 충돌 지점 표시 |
| 1156~1180 | 충돌 계산 | 두 라인 교차점 계산 |
| 1181~1206 | 결과 내보내기 | TXT 파일 다운로드 |
| 1207~1247 | LocalStorage | `saveLS()`, `loadLS()` |
| 1248~1340 | 지형 | 강·산·숲 지형 그리기 |
| 1341~1613 | 이벤트 | 마우스 클릭·드래그·우클릭 핸들러 |
| 1614~1701 | 연맹 관리 | 연맹 추가·편집·삭제 |
| 1702~1731 | 도시 관리 | 도시 점유 설정 |
| 1732~1763 | 템플릿 | JSON 불러오기/내보내기 |
| 1764~1883 | 계산 탭 | 도달 시간·충돌 계산 |
| 1884~1979 | 동맹 관계 탭 | 관계 매트릭스 UI |
| 1980~2417 | 이벤트 탭 | 전장 일정·전소 타이머 |
| 2418~2540 | 전소 계산 탭 | 타워 전소 시간 계산 |
| 2541~2636 | **Firebase 동기화** | 실시간 sync 전체 로직 |
| 2637~2697 | 초기화 | `loadLS()` 호출, 첫 렌더 |

---

## 4. 좌표계

```
맵 좌표 (논리)     ↔     캔버스 좌표 (픽셀)
toC(x, y)    →   canvas px
fromC(px, py) →  map coords

ppc = 3  (픽셀 per 칸, 고정)
snapMode = 3 (스냅 단위, 고정)
```

- 모든 저장값(alliances, cities, waypoints)은 **맵 좌표** 기준
- 렌더링 시만 `toC()`로 변환

---

## 5. 자주 수정하는 값

| 항목 | 위치 | 변수/ID |
|------|------|---------|
| 타워 건설 기본시간 | ~L450 `<input id="tBuildMin">` | `value="23"` |
| 타워 번호별 비용 | L533~543 | `TOWER_COST[]` |
| 시멘트 생산량 | L544~652 | `CEMENT_PROD[]` |
| 맵 기본 크기 | L529~531 | `MAP_W=398, MAP_H=404` |
| 타워 영역 크기 | L530 | `TOWER_RANGE=9` (9×9 고정) |
| 점수 상수 | L707~727 | `SCORE_6CITY` 등 |

---

## 6. Firebase 동기화

**설정값 위치:** L2542~2550

```js
const _fbConfig = {
  apiKey: "AIzaSyA3ou5tXy8Sfd6XExU1T1JUAGWc2Ju1J6o",
  databaseURL: "https://ir-simulator-default-rtdb.firebaseio.com",
  projectId: "ir-simulator",
  ...
};
```

**동기화 흐름:**
```
사용자 액션 → saveLS() → syncPush() [400ms debounce]
                              ↓
                    Firebase rooms/{roomId}/state
                              ↓
                    상대방 브라우저 수신 → 화면 업데이트
```

**동기화 대상 데이터:** `alliances`, `cities`, `relations`, `waypoints`, `terrains`

**Firebase 콘솔:** https://console.firebase.google.com/project/ir-simulator

---

## 7. 배포

```bash
# 수정 후 push하면 GitHub Pages 자동 배포
git add index.html
git commit -m "feat: 변경 내용"
git push origin main
# 1~2분 후 https://endvise.github.io/0021_war_map/ 반영
```

---

## 8. 주의사항

- **단일 파일 원칙**: `index.html` 하나에 전부 있음. 분리하지 말 것
- **빌드 없음**: npm, webpack, 컴파일 없음
- **Firebase 규칙**: 현재 public read/write (전장코드 공유 모델)
- **LocalStorage 키**: `war_map_v2` (다른 키로 바꾸면 기존 데이터 손실)
- **캔버스 재렌더**: 상태 변경 후 반드시 `drawAll()` 호출

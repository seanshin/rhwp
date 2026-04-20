# Task #217 — 2단계 완료보고서

## 팬 + 핀치 줌 활성화 ✅

### 작업 내용

`PagedScrollUIView`에 UIScrollView 줌 기능과 줌 좌표계 처리를 추가했다.

### 핵심 구현

| 항목 | 구현 |
|------|------|
| `viewForZooming` | `containerView` 반환 (페이지 전체 컨테이너 줌) |
| `minimumZoomScale` | `fitScale` (뷰포트 너비 / 문서 최대 너비) |
| `maximumZoomScale` | `max(fitScale * 5, 5.0)` — fit 대비 5배 또는 절대 5배 |
| `bouncesZoom` | `true` — 줌 한계 도달 시 바운스 |
| 초기 줌 | `zoomScale = fitScale` — 화면 폭에 맞춰 로드 |

### 좌표계 처리

**문제**: UIScrollView가 줌되면 `contentOffset`/`bounds`는 스크린 좌표지만, `pageFrames`는 원본(container) 좌표.

**해결**:
- `layoutPagesIfNeeded()`: 가시 영역을 `zoom`으로 나눠 container 좌표로 변환 → 정확한 페이지 교차 판정
- 현재 페이지 추적: `centerY = (contentOffset.y + bounds.height/2) / zoom`

### 중앙 정렬

**문제**: 줌 레벨이 낮을 때 contentSize가 뷰포트보다 작으면 콘텐츠가 좌측 정렬됨.

**해결**: `adjustContentInset()` — 뷰포트와 scaled contentSize 차이의 절반을 `contentInset.left/right`에 적용.

- `scrollViewDidZoom`에서 호출 (줌 중 실시간 조정)
- `layoutSubviews`에서 호출 (회전/크기 변경 대응)

### 팬 (드래그)

UIScrollView 기본 기능 사용. 줌 > 1 상태에서 자동으로 한 손가락 팬 활성화.

### 미구현 (후속 단계)

| 항목 | 단계 |
|------|------|
| 더블탭 줌 (fit ↔ 2배 토글) | 3단계 |
| 줌 종료 시 벡터 재렌더링 | 3단계 |
| DocumentView 연결 | 4단계 |

### 검증 결과

- Xcode 빌드 (iPad Simulator): ✅ BUILD SUCCEEDED
- 시각 검증은 4단계(DocumentView 교체 후)에서 수행

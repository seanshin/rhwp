# Task #217 — 3단계 완료보고서

## 더블탭 줌 + 줌 종료 시 벡터 재렌더링 ✅

### 작업 내용

1. **더블탭 제스처**: 한 손가락 두 번 탭으로 fit ↔ 2.5배 토글
2. **벡터 재렌더링**: 줌 종료 시 `contentScaleFactor`를 `UIScreen.scale × zoomScale`로 갱신 → Core Graphics가 고해상도로 재그리기

### 더블탭 로직

```swift
if zoomScale > fitScale * 1.01 {
    // 확대 상태 → fit으로 축소
    setZoomScale(fitScale, animated: true)
} else {
    // fit 상태 → 탭한 위치 중심으로 2.5배 확대
    let tapPoint = gesture.location(in: containerView)
    let rect = zoomRect(forScale: fitScale * 2.5, center: tapPoint)
    zoom(to: rect, animated: true)
}
```

**`zoomRect(forScale:center:)`**: 줌할 크기 = `bounds / scale`, 중심을 탭 위치로 → 해당 영역을 뷰포트 전체에 맞춤.

### 벡터 재렌더링 전략

**줌 중**: UIScrollView 기본 비트맵 스케일링 (빠름, 품질은 저하됨)
**줌 종료 후**: `scrollViewDidEndZooming` → `updatePageContentScales()`

```swift
let targetScale = UIScreen.main.scale * zoomScale
for (_, pageView) in pageViews {
    pageView.contentScaleFactor = targetScale
    pageView.layer.contentsScale = targetScale
    pageView.setNeedsDisplay()  // draw(_ rect:) 재호출
}
```

- Retina 해상도 × 현재 줌 스케일만큼 해상도 증가
- Core Graphics가 더 많은 픽셀에 그려 벡터 선명도 복원

### 새로 로드되는 페이지도 현재 줌 스케일 반영

`layoutPagesIfNeeded`의 로드 섹션에서 새 `PageCanvasUIView` 생성 시 `contentScaleFactor`를 현재 줌에 맞춰 초기화.

### 미구현 (후속 단계)

| 항목 | 단계 |
|------|------|
| DocumentView 연결 | 4단계 |
| 시뮬레이터/실기기 검증 | 4단계 |

### 검증 결과

- Xcode 빌드 (iPad Simulator): ✅ BUILD SUCCEEDED
- 시각/인터랙션 검증은 4단계(DocumentView 교체 후)에서 수행

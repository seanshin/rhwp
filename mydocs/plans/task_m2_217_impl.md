# Task #217 — 구현계획서

## 문서 뷰 팬 & 줌 기능 (UIScrollView 기반)

### 설계 개요

SwiftUI `ScrollView + LazyVStack`을 UIKit `UIScrollView`로 교체한다. UIScrollView는 팬/줌을 기본 제공하므로 직접 구현하지 않는다.

**구조:**

```
UIScrollView (세로 스크롤 + 확대/축소)
└── UIView (documentContainer, contentSize = 모든 페이지 총 높이)
    ├── PageCanvasUIView (page 0)
    ├── PageCanvasUIView (page 1)
    └── ... (lazy 로드/언로드)
```

**줌 처리 전략:**
- 줌 중: `UIScrollView`가 비트맵 스케일링으로 즉시 반영 (CGContext 재그리기 안함)
- 줌 종료 시: `scrollViewDidEndZooming` → `PageCanvasUIView.setNeedsDisplay()` → 벡터 재렌더링

### 구현 단계 (4단계)

---

#### 1단계: `PagedScrollView` 기본 구조 — 다중 페이지 세로 스크롤

**1-1. `PagedScrollView.swift` 신규 작성 (UIViewRepresentable)**

```swift
struct PagedScrollView: UIViewRepresentable {
    @ObservedObject var viewModel: DocumentViewModel

    func makeUIView(context: Context) -> PagedScrollUIView { ... }
    func updateUIView(_ uiView: PagedScrollUIView, context: Context) { ... }
}

class PagedScrollUIView: UIScrollView, UIScrollViewDelegate {
    private let contentView = UIView()
    private var pageViews: [Int: PageCanvasUIView] = [:]
    private var pageFrames: [CGRect] = []

    func reload(pageCount: Int, pageSizes: [(w: Double, h: Double)], document: RhwpDocument?)
    func scrollViewDidScroll(_ scrollView: UIScrollView) // lazy 로드/언로드
}
```

페이지 배치:
- 각 페이지를 세로로 쌓음 (spacing: 12)
- 뷰포트 너비에 맞춰 초기 스케일 계산 → `zoomScale`로 적용
- `contentSize` = (max 페이지 너비, 모든 페이지 높이 합계)

Lazy 로드:
- `scrollViewDidScroll`에서 가시 영역 + 앞뒤 1페이지를 유지
- 범위 밖 페이지의 `PageCanvasUIView`는 제거 (메모리 보호)

**검증**: 실기기 설치 후 페이지 스크롤 동작 확인 (줌 없이)

---

#### 2단계: 팬 + 핀치 줌 활성화

**2-1. UIScrollView 줌 설정**

```swift
minimumZoomScale = fitScale    // 뷰포트 맞춤
maximumZoomScale = 5.0
delegate = self

func viewForZooming(in scrollView: UIScrollView) -> UIView? {
    return contentView
}
```

**2-2. 초기 `zoomScale` = `fitScale`** (화면 너비 / 페이지 너비)

**2-3. 줌 상태 유지** (페이지 로드 시 페이지 경계 유지)

**검증**: 핀치 줌 동작, 드래그 팬 동작

---

#### 3단계: 더블탭 줌 + 줌 종료 시 벡터 재렌더링

**3-1. 더블탭 제스처**

```swift
let doubleTap = UITapGestureRecognizer(target: self, action: #selector(handleDoubleTap(_:)))
doubleTap.numberOfTapsRequired = 2

@objc func handleDoubleTap(_ gesture: UITapGestureRecognizer) {
    let location = gesture.location(in: contentView)
    if zoomScale > fitScale {
        setZoomScale(fitScale, animated: true)  // fit으로 축소
    } else {
        zoom(to: zoomRect(for: 2.0, center: location), animated: true)  // 2배 확대
    }
}
```

**3-2. 줌 종료 시 벡터 재렌더링**

```swift
func scrollViewDidEndZooming(_ scrollView: UIScrollView, with view: UIView?, atScale scale: CGFloat) {
    for (_, pageView) in pageViews {
        pageView.contentScaleFactor = UIScreen.main.scale * scale
        pageView.setNeedsDisplay()
    }
}
```

Retina 해상도 × 현재 줌 스케일로 `contentScaleFactor` 조정 → Core Graphics가 고해상도로 재렌더링.

**검증**: 확대 후 선명도 확인 (확대해도 글자가 픽셀화되지 않음)

---

#### 4단계: 통합 + 기존 뷰 교체 + 실기기/시뮬레이터 검증

**4-1. DocumentView 수정**

`pageScrollView` 구현체를 `PagedScrollView(viewModel: viewModel)`로 교체.

**4-2. DocumentViewModel 조정**

- `pageTrees: [Int: RenderNode]` 캐시는 유지
- `loadPage` / `unloadPage`는 UIScrollView 델리게이트에서 호출

**4-3. 검증**

- iPad Simulator: 핀치 줌, 드래그 팬, 더블탭 줌
- iPhone 12 Pro 실기기: 동일
- 줌 후 선명도
- 메모리 사용량 (100페이지 스크롤 시)

---

### 파일 변경 목록

| 파일 | 변경 | 단계 |
|------|------|------|
| `rhwp-ios/Sources/PagedScrollView.swift` | 신규 — UIScrollView 래퍼 + 페이지 관리 | 1 |
| `rhwp-ios/Sources/PageCanvasView.swift` | `PageCanvasUIView`를 public으로 노출 (이미 그러함) | 1 |
| `rhwp-ios/Sources/DocumentView.swift` | pageScrollView → PagedScrollView 교체 | 4 |
| `rhwp-ios/Sources/DocumentViewModel.swift` | loadPage/unloadPage 호출부 정리 | 4 |

### M2 범위에서 제외 (후속 이관)

| 항목 | 이관 |
|------|------|
| 페이지 단위 보기 (full-screen single page) | M3 |
| 미니맵 / 썸네일 사이드바 | #100 |
| 회전 잠금 | M3 이후 |

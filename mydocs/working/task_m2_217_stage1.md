# Task #217 — 1단계 완료보고서

## PagedScrollView 기본 구조 + 다중 페이지 스크롤 ✅

### 작업 내용

`rhwp-ios/Sources/PagedScrollView.swift` 신규 작성 — UIScrollView 기반 다중 페이지 세로 스크롤 뷰.

### 구조

```
PagedScrollView (SwiftUI UIViewRepresentable)
└── PagedScrollUIView (UIScrollView + UIScrollViewDelegate)
    └── containerView (UIView, contentSize 설정)
        ├── PageCanvasUIView (page 0)
        ├── PageCanvasUIView (page 1)
        └── ... (lazy 로드/언로드)
```

### 핵심 구현

| 항목 | 구현 |
|------|------|
| 페이지 frame 계산 | `reload()` — 모든 페이지 세로 배치 + 가로 중앙 정렬 |
| `contentSize` | (max 페이지 너비, 모든 페이지 높이 합계) |
| Lazy 로드 | `scrollViewDidScroll` → 가시 영역 ± bounds.height/2 범위 |
| 메모리 관리 | 범위 밖 페이지 `removeFromSuperview()` + `vm.unloadPage()` |
| 페이지 그림자 | `layer.shadowColor/Opacity/Offset/Radius` |
| 현재 페이지 추적 | 가시 영역 중심 페이지 → `vm.currentPage` 갱신 |

### 미구현 (후속 단계)

| 항목 | 단계 |
|------|------|
| 핀치 줌 | 2단계 |
| 드래그 팬 | 2단계 (UIScrollView 기본 제공) |
| 더블탭 줌 | 3단계 |
| 줌 종료 시 벡터 재렌더링 | 3단계 |
| DocumentView 연결 | 4단계 |

### 검증 결과

- Xcode 빌드 (iPad Simulator): ✅ BUILD SUCCEEDED
- 시각 검증은 4단계(DocumentView 교체 후)에서 수행

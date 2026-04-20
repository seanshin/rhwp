# Task #217 — 최종 완료보고서

## 문서 뷰 팬 & 줌 기능 (핀치 줌 + 드래그 팬)

### 목표

사용자가 표준 iOS 제스처로 문서를 확대/이동할 수 있도록 한다.

**변경 전**: SwiftUI ScrollView + LazyVStack — 뷰포트 맞춤 고정 스케일
**변경 후**: UIScrollView 기반 — 핀치/더블탭 줌 + 드래그 팬 + 벡터 재렌더링

### 단계별 결과

| 단계 | 내용 | 결과 |
|------|------|------|
| 1 | PagedScrollView 기본 구조 | ✅ UIScrollView + 다중 페이지 + lazy 로드/언로드 |
| 2 | 팬 + 핀치 줌 활성화 | ✅ `viewForZooming`, `fitScale` 계산, 줌 좌표계 처리, 중앙 정렬 |
| 3 | 더블탭 줌 + 벡터 재렌더링 | ✅ fit ↔ 2.5배 토글, `scrollViewDidEndZooming` → `contentScaleFactor × zoom` |
| 4 | DocumentView 통합 + 검증 | ✅ iPad Simulator + iPhone 12 Pro 실기기 |

### 핵심 설계

**UIScrollView 기반 (선택지 A)**
- `viewForZooming` → `containerView` 반환 → UIScrollView 기본 줌 활용
- 제스처 충돌 자동 해결 (세로 스크롤 vs 핀치 vs 팬)
- iOS 표준 바운스/감속 UX 그대로

**줌 성능 전략**
- **줌 중**: UIScrollView 기본 비트맵 스케일링 (60fps 유지)
- **줌 종료 후**: Core Graphics 벡터 재렌더링 (선명도 복원)

```swift
func scrollViewDidEndZooming(_ scrollView: UIScrollView, with view: UIView?, atScale scale: CGFloat) {
    let targetScale = UIScreen.main.scale * zoomScale
    for (_, pageView) in pageViews {
        pageView.contentScaleFactor = targetScale
        pageView.setNeedsDisplay()
    }
}
```

**좌표계 처리**
- `pageFrames`: container 원본 좌표
- `contentOffset`/`bounds`: UIScrollView 좌표 (줌 반영됨)
- 변환: `container_rect = scroll_rect / zoomScale`

### 생성/변경 파일

| 파일 | 역할 |
|------|------|
| `rhwp-ios/Sources/PagedScrollView.swift` (신규) | UIScrollView 래퍼 + 다중 페이지 + 팬/줌 + 더블탭 |
| `rhwp-ios/Sources/DocumentView.swift` | pageScrollView 구현체 교체 |

### 검증 결과

- iPad Simulator (iPad Pro 11-inch M4): ✅ 전체 기능 동작
- iPhone 12 Pro 실기기: ✅ 전체 기능 동작
- Xcode 빌드: ✅
- 기존 렌더링 회귀 없음 (#93 결과 유지)

### 후속 작업

- #100 iPad 뷰어 UX 개선 (네비게이션, 툴바, 썸네일 사이드바 등)
- M3: Metal 가속, 페이지 단위 보기 모드, 접근성

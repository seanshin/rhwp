# Task #217 — 수행계획서

## 문서 뷰 팬 & 줌 기능 (핀치 줌 + 드래그 팬)

### 배경

#93에서 Core Graphics 네이티브 렌더러로 전환했다. iPhone 실기기에서 확인한 결과:

- 뷰포트에 맞춰 자동 축소되지만 **고정 스케일**이다
- 세밀한 내용(작은 글자, 표 셀 내용)을 확대해서 볼 수 없다
- 기본적인 문서 뷰어에서 기대하는 팬/줌 상호작용이 부재하다

현재 임시 수정으로 `GeometryReader` + `scaleEffect` + 중앙 정렬만 적용된 상태다.

### 목표

사용자가 직관적으로 문서를 확대/이동하며 볼 수 있도록 표준 iOS 제스처를 구현한다.

1. **핀치 줌**: 두 손가락으로 확대/축소
2. **드래그 팬**: 확대된 상태에서 손가락으로 이동
3. **더블탭 줌**: 빠른 확대/축소 토글 (fit ↔ 2배)
4. **줌 레벨 제한**: 최소 fit 스케일, 최대 4~5배
5. **페이지 간 스크롤 유지**: 확대 상태에서도 세로 스크롤로 페이지 이동 가능

### 범위

**포함:**
- 다중 페이지 스크롤 뷰에서의 팬/줌
- 핀치/더블탭 제스처 처리
- 벡터 렌더링의 장점을 활용한 고해상도 유지 (재렌더링으로 선명도 확보)

**제외:**
- 페이지 1장씩 선택하여 독립 줌 (페이지 단위 뷰어)
- 회전 잠금/해제
- 미니맵/오버뷰

### 기술 검토

**선택지 A: UIScrollView 기반**
- `UIScrollView` + `viewForZooming` 위임 활용 — iOS 표준 줌/팬
- SwiftUI `UIViewRepresentable`로 래핑
- 페이지 간 스크롤과 줌 상태의 조화가 명확
- **권장**

**선택지 B: SwiftUI MagnificationGesture + DragGesture**
- 순수 SwiftUI 구현
- 제스처 충돌 해결에 추가 로직 필요 (동시 제스처, 스크롤뷰와의 상호작용)
- 구현 복잡도 높음

A안 권장. 기존 `ScrollView + LazyVStack` 구조를 `UIScrollView`로 전환한다.

### 성능 고려사항

Core Graphics 벡터 렌더링이므로 줌 변경 시 `setNeedsDisplay()` 호출로 고해상도 재렌더링 가능. 다만:

- 줌 제스처 중 매 프레임 재렌더링하면 끊길 수 있음 → **줌 중엔 비트맵 스케일링, 줌 종료 시 재렌더링** 패턴 필요
- `contentsScale` 조정으로 Retina 해상도 유지

### 위험 요소

| 위험 | 대응 |
|------|------|
| LazyVStack 메모리 관리와 UIScrollView 재사용 방식 불일치 | UIScrollView로 전환 시 페이지 로드/언로드 로직 재구성 |
| 줌 중 성능 저하 | 줌 중 비트맵, 줌 종료 후 벡터 재렌더링 |
| 제스처 충돌 (핀치 vs 세로 스크롤) | UIScrollView 기본 동작으로 자동 해결 |

### 산출물

| 파일 | 내용 |
|------|------|
| `rhwp-ios/Sources/PagedScrollView.swift` (신규) | UIScrollView 기반 다중 페이지 + 팬/줌 |
| `rhwp-ios/Sources/DocumentView.swift` | ScrollView + LazyVStack → PagedScrollView 교체 |
| `rhwp-ios/Sources/DocumentViewModel.swift` | 페이지 로드/언로드 API 재조정 |
| `mydocs/working/task_m2_217_stage*.md` | 단계별 완료보고서 |
| `mydocs/report/task_m2_217_report.md` | 최종 완료보고서 |

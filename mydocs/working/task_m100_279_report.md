# 최종 결과 보고서 — Task #279

**이슈**: [#279 목차 right tab 리더 점 렌더링 + 소제목 페이지 번호 정렬 불일치](https://github.com/edwardkim/rhwp/issues/279)  
**마일스톤**: v1.0.0 (M100)  
**브랜치**: `local/task279`  
**PR**: [#282](https://github.com/edwardkim/rhwp/pull/282)  
**완료일**: 2026-04-24

---

## 1. 작업 요약

KTX.hwp 목차 페이지의 두 가지 렌더링 버그를 수정하였다.

---

## 2. 수정 내용

### 버그 1: fill_type=3 리더 도트 렌더링

**파일**: `src/renderer/svg.rs`, `src/renderer/web_canvas.rs`

| | 수정 전 | 수정 후 |
|---|---|---|
| SVG 출력 | `stroke-width="0.5" stroke-dasharray="1 2"` (사각 대시) | `stroke-width="1.0" stroke-dasharray="0.1 3" stroke-linecap="round"` (원형 점) |
| WASM Canvas | `draw_line(ly, 0.5, &[1.0, 2.0])` | `set_line_cap("round")` + `draw_line(ly, 1.0, &[0.1, 3.0])` |

### 버그 2: 소제목 페이지번호 탭 위치 불일치

**파일**: `src/renderer/layout/text_measurement.rs` L71

**근본 원인**:
- `TabStop.position` = 단(column) 기준 절대 px
- `available_width = col_width - effective_margin_left - margin_right` (문단 상대값)
- 들여쓰기(left_margin > 0)가 있는 소제목(ps_id=111)에서 `available_width`가 작아짐
- 오른쪽 탭(type=1)이 잘못 클램핑되어 페이지번호가 좌측으로 밀림

**수정**: type=1(오른쪽) 탭은 클램핑 제외

```rust
// 수정 전:
if ts.position > available_width && available_width > 0.0 {

// 수정 후:
if ts.tab_type != 1 && ts.position > available_width && available_width > 0.0 {
```

**결과**:

| 항목 | 수정 전 | 수정 후 |
|------|---------|---------|
| 소제목 페이지번호 x | ~700px | 717.9px |
| 장제목 페이지번호 x | 717.5px | 717.5px |
| 차이 | ~17px | **0.4px** ✅ |

---

## 3. 검증

- `cargo test`: 793 passed, 0 failed ✅
- KTX.hwp 목차 SVG: 원형 점 리더 + 페이지번호 정렬 확인 ✅
- 웹 에디터(weve.io.kr): 배포 후 시각 확인 ✅

---

## 4. 이전 revert와의 차이

commit `78330fd`에서 revert된 수정은 `ts.available_width`를 변경하여 text wrapping에 영향을 주었다. 이번 수정은 클램핑 조건(`tab_type != 1`)만 추가하여 wrapping 로직은 전혀 변경하지 않는다.

---

## 5. 관련 이슈

- Issue #267 (right tab 선행 공백) — 관련 선행 수정 (PR #273)
- Issue #274 — 이번 수정으로 해결됨 (superseded)

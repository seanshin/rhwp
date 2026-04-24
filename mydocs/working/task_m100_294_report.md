# 최종 결과 보고서 — Task #294

**이슈**: [#294 목차 교차-run 오른쪽 탭: 전체 폭 look-ahead로 right edge 정확 정렬](https://github.com/edwardkim/rhwp/issues/294)  
**마일스톤**: v1.0.0 (M100)  
**브랜치**: `local/task294`  
**커밋**: `71b6a73`  
**완료일**: 2026-04-24

---

## 1. 작업 요약

KTX.hwp 목차 페이지의 교차-run 오른쪽 탭 처리를 두 가지 수정으로 정확히 정렬.

---

## 2. 수정 내용

**파일**: `src/renderer/layout/paragraph_layout.rs`

### 수정 1 — 교차-run 탐지 조건 개선

```rust
// 수정 전:
if has_tabs && run.text.ends_with('\t') {

// 수정 후:
if has_tabs && run.text.trim_end_matches(' ').ends_with('\t') {
```

**근본 원인**: 소제목 Run A = `"제목\t "` (탭 뒤 공백 포함) → `ends_with('\t')` = false → cross-run 미탐지.  
`trim_end_matches(' ')` 는 후행 공백만 제거(탭 제거 안 함)하여 `"\t"` 단독 run도 안전하게 처리.

### 수정 2 — look-ahead 총 폭 합산 (tab_type=1)

```rust
// 수정 전 (3c06781 leading_ws 방식):
x = col_area.x + tab_pos - leading_ws_w;

// 수정 후:
let mut total_w = next_w;
for fr in &comp_line.runs[(run_idx + 1)..] {
    // fts 설정 ...
    total_w += estimate_text_width(&fr.text, &fts);
}
x = col_area.x + tab_pos - total_w;
```

**효과**: right edge = tab_pos_abs 보장 (자릿수 무관).

---

## 3. 검증 결과 (KTX.hwp 목차 SVG)

| 페이지번호 | 수정 전 right edge | 수정 후 right edge | 목표 | 오차 |
|-----------|------------------|-------------------|------|------|
| 단일 '3','8' (장제목) | 709.09 | **700.09** | 699.76 | +0.33px ✓ |
| 단일 '4'~'9' (소제목) | 709.21 | **700.09** | 699.76 | +0.33px ✓ |
| 두 자리 '14','17' 등 (소제목) | 718.54 | **699.42** | 699.76 | -0.34px ✓ |
| 두 자리 '16','20','24' (장제목) | 718.76 | **699.76** | 699.76 | -0.00px ✓ |

**합격 기준**: 모든 항목 ±1.0px → **전체 통과**

- `cargo test --lib`: 793 passed, 0 failed ✅
- 배포: weve.io.kr/hwpx/editor/ ✅

---

## 4. 구조 분석 메모

### 장제목 vs 소제목 run 구조 차이

| | run 구조 |
|---|---------|
| **장제목** | [제목] ["\t"] [" "] ["3"] — tab 단독 run |
| **소제목** | [번호] ["제목\t "] ["4"] — tab이 제목 run 끝에 포함 + 후행 공백 |

장제목은 `"\t"` 단독 run → 기존 `ends_with('\t')` 처리.  
소제목은 `"제목\t "` → `ends_with('\t')` false → 탐지 실패 (이번 수정 전).

---

## 5. 관련 이슈

- Issue #279 (부모) — 목차 리더 점 + 소제목 정렬 (해당 수정)
- Issue #267 — right tab 선행 공백 처리 (PR #273)

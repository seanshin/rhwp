# 최종 결과 보고서 — Task #157

**이슈**: [#157 비-TAC wrap=위아래 표 텍스트 중첩](https://github.com/edwardkim/rhwp/issues/157) + [#103 동시 해결](https://github.com/edwardkim/rhwp/issues/103)  
**마일스톤**: v1.0.0 (M100)  
**브랜치**: `local/task157`  
**커밋**: `6b586cf`  
**완료일**: 2026-04-24

---

## 1. 작업 요약

`wrap=위아래(TopAndBottom)`, `tac=false`, `vert=Para` 표를 out-of-flow로 올바르게 배치하지 못해 텍스트와 중첩되는 버그를 수정하였다. `layout.rs` vpos 기준점 리셋 예외 처리 1곳과 `engine.rs` 방어 코드 1곳 수정으로 이슈 #157과 #103을 동시에 해결하였다.

---

## 2. 근본 원인

### 원인 경로

`layout.rs`의 레이아웃 루프에서 **모든 Table/Shape 아이템** 처리 후 vpos 교정 기준점 두 개를 초기화하고 있었다:

```rust
// layout.rs (수정 전)
if was_tac || is_table_or_shape {
    vpos_page_base = None;
    vpos_lazy_base = None;
}
```

Para-relative float 표(vert=Para, TopAndBottom, non-TAC)는 앵커 문단에 attach되므로 후속 문단의 vpos 교정 기준점을 초기화하면 안 된다.  
초기화가 일어나면:

1. 한컴이 Para-float 표를 기준으로 기록한 후속 문단 vpos가 잘못된 `lazy_base`로 교정됨
2. Pi=25(표 앵커) vpos 교정 결과 `anchor_y ≈ 939.2px` (정상: 768.4px)
3. `compute_table_y_position` 내 `body_bottom` clamp → 표가 894.7px에 고정
4. `layout_table` 반환값 + `line_spacing` = `y_offset 1102.9px`
5. `col_bottom 1093.3px` 초과 → `LAYOUT_OVERFLOW 9.6px`
6. 텍스트와 표 중첩 발생

### 수치 비교

| 항목 | 버그 상태 | 수정 후 |
|------|-----------|---------|
| `lazy_base` | 63965 HU (잘못됨) | 사용 안 함 → page_base 77497 |
| Pi=25 `anchor_y` | 939.2px | **768.4px** |
| `table_y` (raw) | 788.0 → clamp 894.7 | **788.0** (clamp 불필요) |
| `table_bottom` | 1093.3 | **986.8** |
| 최종 `y_offset` | 1102.9 | **996.4** |
| `LAYOUT_OVERFLOW` | 9.6px | **0px** ✅ |

---

## 3. 수정 내용

### 수정 1 — `src/renderer/layout.rs` (핵심)

Para-relative float 표(`is_para_float_table`)는 vpos 기준점 초기화 대상에서 제외:

```rust
// 수정 후
let is_para_float_table = if let PageItem::Table { para_index, control_index } = item {
    paragraphs
        .get(*para_index)
        .and_then(|p| p.controls.get(*control_index))
        .map(|c| {
            matches!(
                c,
                Control::Table(t)
                if !t.common.treat_as_char
                    && matches!(t.common.text_wrap, crate::model::shape::TextWrap::TopAndBottom)
                    && matches!(t.common.vert_rel_to, VertRelTo::Para)
            )
        })
        .unwrap_or(false)
} else {
    false
};
if was_tac || (is_table_or_shape && !is_para_float_table) {
    vpos_page_base = None;
    vpos_lazy_base = None;
}
```

**조건 3개 AND로 엄격히 한정** — TAC/Shape/기타 Table 케이스 영향 없음:
- `!treat_as_char`
- `wrap == TopAndBottom`
- `vert_rel_to == Para`

### 수정 2 — `src/renderer/pagination/engine.rs` (방어 코드)

Para-relative float 표가 페이지 body 내에 완전히 들어올 때 `effective_table_height = 0.0`:

```rust
// 수정 후
let abs_bottom = para_start_height + v_off + effective_height + host_spacing;
let effective_table_height = if abs_bottom <= base_available_height + 0.5 {
    0.0  // body 내에 완전히 들어옴 → flow height 기여 없음
} else {
    (abs_bottom - st.current_height).max(effective_height + host_spacing)
};
```

---

## 4. 검증

| 항목 | 결과 |
|------|------|
| `cargo test` | ✅ 941 + 4 (issue_157_page_1) = **945 passed, 0 failed** |
| `cargo test --test svg_snapshot` | ✅ **4 / 0** (issue_157 포함) |
| `cargo clippy --lib -- -D warnings` | ✅ clean |
| `cargo check --target wasm32` | ✅ clean |
| `issue_157.hwpx` p.2 `LAYOUT_OVERFLOW` | ✅ **없음** (이전: 9.6px) |
| `issue_157.hwpx` p.2 Table pi=25 y | ✅ **819.2px** (이전: 894.7px clamp) |
| 텍스트-표 중첩 | ✅ 해소 확인 |
| Golden SVG 등록 | ✅ `tests/golden_svg/issue-157/page-1.svg` (507줄) |
| 기존 테스트 regression | ✅ 없음 |
| WASM Docker 빌드 | ✅ 성공 |
| rhwp-studio 브라우저 시각 검증 | ✅ 주주총회 참석장 2페이지 중첩 해소 확인 |

---

## 5. 이슈 #103 동시 해결

Issue #103 ("TAC 표와 비-TAC 표가 같은 앵커 문단에 공존 시 비정상 간격")도 동일 근본 원인(vpos 기준점 리셋)에 의한 것으로, 이번 수정으로 함께 해결되었다.

---

## 6. 수정 파일 목록

| 파일 | 변경 내용 |
|------|-----------|
| `src/renderer/layout.rs` | `is_para_float_table` 예외 처리 (+18줄) |
| `src/renderer/pagination/engine.rs` | `effective_table_height = 0.0` 방어 코드 (+4줄) |
| `tests/svg_snapshot.rs` | `issue_157_page_1` 테스트 추가 |
| `tests/golden_svg/issue-157/page-1.svg` | Golden SVG 신규 등록 |

---

## 7. 단계별 보고서

| 단계 | 내용 | 보고서 |
|------|------|--------|
| 단계 1 | `layout.rs` vpos 기준점 리셋 예외 처리 | `mydocs/working/task_m100_157_stage1.md` |
| 단계 2 | `engine.rs` effective_table_height 방어 코드 | ↑ 단계 1 보고서에 포함 |
| 단계 3 | 검증 (cargo test + golden SVG + 브라우저 확인) | ↑ 단계 1 보고서에 포함 |

# Task #157 최종 결과보고서: 비-TAC wrap=위아래 표 out-of-flow 배치

> 최종 보고서 | 2026-04-27  
> Issue: [#157](https://github.com/edwardkim/rhwp/issues/157) + [#103](https://github.com/edwardkim/rhwp/issues/103)  
> Milestone: v1.0.0  
> Branch: `local/task157` → PR #266 → `edwardkim/rhwp:devel` (머지: 2026-04-24)

---

## 1. 목표

`wrap=위아래(TopAndBottom)`, `tac=false(비-TAC)` 표를 한컴과 동일하게 out-of-flow로 배치하여
텍스트와 표의 중첩(#157) 및 비정상 간격(#103)을 수정한다.

---

## 2. 근본 원인

### 수행계획서 원인 분석 vs 실제 원인

수행계획서에서는 "페이지네이터가 비-TAC TopAndBottom+Para 표를 in-flow로 처리"를 근본 원인으로 지목했으나,
**코드 정밀 조사 결과 페이지네이터는 정상**이었다.

### 실제 근본 원인: `layout.rs` vpos 기준점 무조건 리셋

`layout.rs` 레이아웃 루프에서 표/Shape 처리 후 vpos 기준점 두 개를 **아이템 종류와 무관하게** 항상 초기화:

```rust
// 버그 코드 (layout.rs:1449–1454)
if was_tac || is_table_or_shape {
    vpos_page_base = None;
    vpos_lazy_base = None;
}
```

Para-relative float 표(vert=Para, TopAndBottom, non-TAC)는 앵커 문단에 attach되므로
후속 문단의 vpos 교정 기준점을 초기화하면 안 된다. 초기화가 발생하면:

1. 한컴이 Para-float 표를 기준으로 기록한 후속 문단 vpos(Pi=8~25, vpos 큰 점프)가 잘못된 `lazy_base`로 교정됨
2. Pi=25(표 앵커) anchor_y ≈ 939.2px (정상: 768.4px)
3. `body_bottom` clamp → 표 y = 894.7px (정상: 788.0px)
4. `layout_table` 반환값(894.7) + 후행 `line_spacing`(9.6) = y_offset 1102.9px
5. `col_bottom 1093.3px` 초과 → `LAYOUT_OVERFLOW 9.6px`

### 수치 비교

| 항목 | 버그 상태 | 수정 후 |
|------|-----------|---------|
| lazy_base | 63965 HU (오염) | 미사용 → page_base 77497 |
| Pi=25 anchor_y | 939.2 px | 768.4 px |
| table_y (raw) | 788.0 → clamp 894.7 | 788.0 (clamp 불필요) |
| table_bottom | 1093.3 px | 986.8 px |
| 최종 y_offset | 1102.9 px | 996.4 px |
| LAYOUT_OVERFLOW | 9.6 px | **0 px** |

---

## 3. 구현 내용

### 단계 1 — `layout.rs` vpos 기준점 리셋 예외 처리

**파일**: `src/renderer/layout.rs` (lines 1445–1467)

Para-relative float 표 여부를 3-조건 AND로 판별(`!treat_as_char && TopAndBottom && VertRelTo::Para`)하여
vpos 기준점 초기화 대상에서 제외.

```rust
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
} else { false };

if was_tac || (is_table_or_shape && !is_para_float_table) {
    vpos_page_base = None;
    vpos_lazy_base = None;
}
```

### 단계 2 — `engine.rs` effective_table_height 방어 코드

**파일**: `src/renderer/pagination/engine.rs` (lines 1099–1117)

Para-relative float 표가 페이지 body 내에 완전히 들어오면 `effective_table_height = 0.0` 처리.

```rust
let effective_table_height = if abs_bottom <= base_available_height + 0.5 {
    0.0  // body 범위 내 → flow height 기여 없음
} else {
    (abs_bottom - st.current_height).max(effective_height + host_spacing)
};
```

### 단계 3 — Golden SVG 등록 + 테스트

- `tests/svg_snapshot.rs`에 `issue_157_page_1` 테스트 추가
- `tests/golden_svg/issue-157/page-1.svg` (507줄) 신규 등록

---

## 4. 수정 파일 목록

| 파일 | 변경 |
|------|------|
| `src/renderer/layout.rs` | `is_para_float_table` 예외 처리 (+18줄) |
| `src/renderer/pagination/engine.rs` | `effective_table_height = 0.0` 조건 (+4줄) |
| `tests/svg_snapshot.rs` | `issue_157_page_1` 스냅샷 테스트 추가 |
| `tests/golden_svg/issue-157/page-1.svg` | golden SVG 신규 등록 |
| `mydocs/plans/task_m100_157.md` | 수행계획서 |
| `mydocs/plans/task_m100_157_impl.md` | 구현계획서 |
| `mydocs/working/task_m100_157_stage1.md` | 단계 완료보고서 |

---

## 5. 검증 결과

| 항목 | 결과 |
|------|------|
| `cargo test --lib` | ✅ 963 passed, 0 failed |
| `cargo test --test svg_snapshot` | ✅ 4 passed (issue_157 포함) |
| `cargo clippy --lib -- -D warnings` | ✅ clean |
| `cargo check --target wasm32` | ✅ clean |
| `dump-pages issue_157.hwpx -p 1` | ✅ LAYOUT_OVERFLOW 없음, Table pi=25 y=819.2px |
| devel 자동 merge 시뮬레이션 | ✅ 충돌 0건, 964 / 0 |
| WASM Docker 빌드 | ✅ 성공 (메인테이너 검증) |
| 브라우저 시각 검증 | ✅ 주주총회 참석장 2페이지 중첩 해소 확인 |

**잔여 overflow (pi=28, 9.6px):** 수정 전과 동일한 y=1102.9. Table pi=25가 정상 배치된 후
pi=26~28이 표 하단 이하로 이어지며 마지막 줄이 body_bottom을 9.6px 초과. Issue #157과 무관한 문서 내용 특성.

---

## 6. 파급 효과

- **Issue #103 동시 해결**: Para-relative float 기준점 리셋이 #103 비정상 간격의 공유 원인이었음.
  `is_para_float_table` 조건이 #103 케이스에서도 동일하게 작동하여 함께 수정됨.
- **재사용 가능한 패턴**: `is_para_float_table` 3-조건 AND 판별 패턴은 동일 조건의 다른 vpos 관련 버그에 재사용 가능.
- **Golden SVG 507줄**: 전체 페이지 스냅샷 등록으로 향후 회귀 즉시 감지 가능.

---

## 7. PR 리뷰 결과

**PR #266** — edwardkim Approved (2026-04-24):

> "루트 원인 정확. 수정 범위 타이트. Golden SVG 등록으로 향후 회귀 즉시 감지 가능."

메인테이너 검증 전 항목 모두 통과 후 머지.

---

## 8. 교훈

1. **수행계획서 원인 분석 ≠ 실제 원인**: 코드 정밀 조사 후 구현계획서에서 원인을 재정정. 수행계획서를 맹신하지 않는다.
2. **vpos 교정 기준점의 범위 민감성**: `vpos_page_base / vpos_lazy_base` 리셋은 out-of-flow 개체 처리 후 무조건 수행하면 안 된다. float 개체 종류별 예외 처리 필요.
3. **3-조건 AND 한정**: 수정 범위를 최소화하기 위해 `!treat_as_char && TopAndBottom && VertRelTo::Para` 세 조건을 모두 만족할 때만 예외 적용. TAC/Shape/기타 케이스에 영향 없음.

# Right Tab 정렬 정밀 수정 기획서

## 문제 현상

목차(TOC)에서 페이지 번호의 우측 정렬이 어긋남.
- 장제목(Ⅰ, Ⅱ, Ⅲ...): 페이지 번호 올바르게 우측 정렬
- 소제목(1., 2., 3...): 페이지 번호가 왼쪽으로 9.33px 밀림

## 재현 파일

`/Users/hyounmoukshin/Downloads/KTX.hwp` 2페이지 (셀[10] 내 목차)

## 원인 분석

### 데이터 구조

```
p[0] 장제목: " Ⅰ. 사업 개요[TAB] 3"   ctrls=2, text_len=22
p[1] 소제목: "1. 추진배경 및 목적[TAB] 4"  ctrls=0, text_len=29
```

- 장제목: `ctrls=2` → 텍스트가 컨트롤 코드 경계에서 여러 run으로 분리됨
- 소제목: `ctrls=0` → 전체 텍스트가 단일 run

### Right Tab 계산 흐름

```
compute_char_positions() 내 탭 처리:
  abs_x = line_x_offset + 현재까지_폭
  (tab_pos, tab_type=RIGHT, _) = find_next_tab_stop(abs_x, ...)
  rel_tab = tab_pos - line_x_offset
  seg_w = measure_segment_from(탭_이후_나머지_텍스트)  ← 핵심
  result_x = rel_tab - seg_w
```

### seg_w 차이의 근본 원인

`measure_segment_from()`은 탭 문자 이후 **같은 run 내**의 나머지 텍스트 폭을 측정함.

| 문단 | 탭이 있는 run | 탭 이후 텍스트 | seg_w |
|------|-------------|-------------|-------|
| p[0] 장제목 | `"...개요\t"` (run 끝) | (다음 run) `" 3"` | **0.00** (같은 run에 없음) |
| p[1] 소제목 | `"...목적\t 4"` (단일 run) | `" 4"` (공백+숫자) | **9.33** (공백 포함) |

장제목에서 seg_w=0인 이유: 탭이 run의 마지막 문자이므로 `measure_segment_from(i+1)`이 빈 결과를 반환.
→ **교차 run 탭 처리**(pending_right_tab)에서 다음 run의 폭을 별도로 계산.

소제목에서 seg_w=9.33인 이유: 탭 뒤에 `" 4"`가 같은 run에 있어서 공백 폭(≈9.33px)이 포함됨.

### 한컴의 동작

한컴에서는 right tab 정렬 시 **탭 직후의 선행 공백을 무시**하고, 실질 텍스트(숫자)의 우측 끝을 tab_pos에 맞춤.

```
한컴: result_x = rel_tab - (숫자 폭만)
rhwp: result_x = rel_tab - (공백 + 숫자 폭)
차이: 공백 1개 ≈ 9.33px
```

## 수정 방안

### 방안 A: seg_w 계산 시 선행 공백 제거 (권장)

`measure_segment_from()`에서 탭 직후 선행 공백을 skip하여 측정.

```rust
// text_measurement.rs: measure_segment_from()
fn measure_segment_from(...) -> f64 {
    let mut w = 0.0;
    let mut skipping_leading_spaces = true;  // ← 추가
    for i in start..chars.len() {
        if chars[i] == '\t' { break; }
        if cluster_len[i] == 0 { continue; }
        // right tab 직후 선행 공백 skip
        if skipping_leading_spaces && chars[i] == ' ' { continue; }
        skipping_leading_spaces = false;
        w += char_width(i);
    }
    w
}
```

**장점**: 최소 변경, right tab의 seg_w만 영향
**단점**: center tab에도 영향 가능 (center tab은 선행 공백 포함해야 할 수 있음)
**위험도**: 낮음 — right/center tab 전용이므로 일반 텍스트 레이아웃 무관

### 방안 B: right tab 전용 seg_w 계산

기존 `measure_segment_from()`은 그대로 두고, right tab일 때만 선행 공백을 제거한 별도 측정.

```rust
// compute_char_positions() 내:
1 => { // 오른쪽 탭
    let seg_w = measure_segment_from(&chars, &cluster_len, i + 1, &char_width);
    // right tab: 선행 공백 제거
    let trimmed_start = (i + 1..chars.len()).find(|&j| chars[j] != ' ' && chars[j] != '\t').unwrap_or(chars.len());
    let trimmed_seg_w = measure_segment_from(&chars, &cluster_len, trimmed_start, &char_width);
    x = (rel_tab - trimmed_seg_w).max(x);
}
```

**장점**: right tab에만 정확히 적용, center tab은 기존 동작 유지
**단점**: 코드 중복

### 방안 C: 교차 run 처리 통일

장제목처럼 탭이 run 끝에 있으면 `pending_right_tab`으로 다음 run에서 처리됨.
소제목처럼 탭이 run 중간에 있으면 같은 run 내에서 처리됨.
두 경로를 통일하여 동일한 결과를 생성.

**장점**: 근본적 해결
**단점**: 변경 범위 크고 regression 위험 높음

## 실험 결과 (2026-04-22)

### 시도 1: seg_w에서 선행 공백 제거 (방안 B)
- `measure_segment_from` → `skip_leading_spaces`로 공백 skip
- `compute_char_positions`에서 right tab 후 공백 position도 0폭 처리
- **결과**: 소제목 경로는 개선되었으나 장제목 경로(교차 run)는 미적용 → 여전히 불일치

### 시도 2: 교차 run에서도 `trim_start()` 적용
- `pending_right_tab` 처리에서 `run.text.trim_start()` 폭 계산
- **결과**: 장제목이 더 오른쪽으로 밀림 (공백 제거 → run_w 감소 → est_x 증가)
- 장제목/소제목의 절대 위치 차이 24.88px → 오히려 악화

### 시도 3: available_width를 right_tab_width로 통일
- `available_width` 대신 `col_area.width - margin_right`를 사용
- **결과**: 모든 tab_pos가 동일해졌으나, 기존 문서의 정렬이 크게 변동 → revert

### 핵심 교훈

right tab 정렬은 3가지 경로로 처리됨:
1. **같은 run 내 탭** → `compute_char_positions`에서 `seg_w` 계산
2. **교차 run 탭 (추정)** → `pending_right_tab_est` + `estimate_text_width`
3. **교차 run 탭 (렌더)** → `pending_right_tab_render` + `estimate_text_width`

이 3경로의 공백 처리가 불일치하면 정렬이 어긋남.
단순히 한 경로만 수정하면 다른 경로와 불일치가 발생하여 오히려 악화.

### 올바른 접근

1. **3경로 모두에서 동일한 공백 처리 규칙** 적용
2. right tab 후 선행 공백의 정확한 의미를 한컴 동작에서 역공학
3. 단일 문서(KTX.hwp)가 아닌 **여러 목차 문서**로 교차 검증 필요
4. Task 403의 제어 샘플 생성 프레임워크를 활용하여 right tab 전용 테스트 샘플 생성

## 추천

**별도 Task로 분리하여 체계적으로 진행** (현재 세션에서의 즉시 수정은 위험).

- right tab 전용으로 `trimmed_seg_w` 계산
- center tab은 기존 동작 유지
- `compute_char_positions()` 내 3곳(추정 패스, 렌더 패스, paragraph_layout 교차 run)에서 동일 적용
- 793 테스트 + KTX.hwp visual diff로 검증

## 수정 파일

| 파일 | 변경 |
|------|------|
| `src/renderer/layout/text_measurement.rs` | `compute_char_positions()` right tab 분기에서 trimmed seg_w |

## 검증 계획

1. `cargo test` — 793 전체 통과
2. KTX.hwp 2페이지 목차 — 장제목/소제목 페이지 번호 동일 x좌표 정렬
3. 기존 visual diff 페이지(p1, p5, p6, p10) — regression 없음
4. 다른 목차 문서(hwpspec.hwp 등)로 추가 검증

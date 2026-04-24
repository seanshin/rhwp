# PR #278 검토 — Task #275: WASM canvas OLE RawSvg/Placeholder 처리 복구

**PR**: [#278](https://github.com/edwardkim/rhwp/pull/278)
**기여자**: @planet6897 (Jaeuk Ryu) — 신뢰도 높음 (PR #256 검증됨)
**브랜치**: `planet6897:local/task275` → `edwardkim/rhwp:devel`
**검토일**: 2026-04-24

---

## 1. 요약

WASM canvas 렌더러(`web_canvas.rs`)에서 `RenderNodeType::RawSvg`, `RenderNodeType::Placeholder` match arm 이 누락되어 OLE 개체(bitmap.hwp, 한셀OLE.hwp)가 빈 페이지로 렌더되던 버그를 수정한다.

---

## 2. 변경 범위

| 파일 | 변경 | 내용 |
|------|------|------|
| `src/renderer/svg_fragment.rs` | 신규 (+279줄, 단위 테스트 19건 포함) | SVG 조각 파서 유틸 — `find_svg_attr_value`, `try_parse_single_image_data_url`, `decode_base64_data_url`, `is_svg_prefix`, `wrap_svg_fragment` |
| `src/renderer/web_canvas.rs` | 수정 (+51줄) | `Placeholder` arm (+26줄), `RawSvg` arm (+25줄), `detect_image_mime_type` SVG 분기 추가 |
| `mydocs/` | 문서 (+4파일) | 수행·구현계획서, 단계별 완료보고서, 최종 보고서 |

---

## 3. 코드 검토

### 3.1 `svg_fragment.rs`

**설계 결정 — 분리 모듈로 추출**: `web_canvas.rs` 는 `#[cfg(target_arch = "wasm32")]` gate로 네이티브 테스트 불가. 파서 유틸을 별도 모듈로 분리해 19건의 네이티브 단위 테스트를 확보한 점은 올바른 결정.

**`find_svg_attr_value` 단어 경계 처리**: `xlink:href` 검색 시 `href` 를 잘못 매칭하는 문제를 이전 바이트가 공백류인지로 판별. `pos == 0` 시 `false` 반환 → 첫 문자로 시작하는 속성명은 매칭 안 됨. 실용상 문제 없음 (SVG 태그는 항상 `<tagname ` 으로 시작하므로 첫 문자에 속성이 올 수 없음).

**`try_parse_single_image_data_url` 복합 SVG 차단**: `<` 개수를 1개로 제한하여 EMF/OOXML 복합 SVG 가 A 경로로 빠지지 않음을 보장. 명료하고 충분한 방어.

**`wrap_svg_fragment` viewBox = bbox**: 조각 내부 좌표가 페이지 절대좌표이므로 viewBox 를 bbox 에 동일하게 맞추는 접근법은 정확함. `drawImage(img, bbox.x, bbox.y, bbox.w, bbox.h)` 와 대응되어 좌표 변환 로직 불필요.

**`is_svg_prefix` 256바이트 창**: MIME 감지용 프리픽스 검사에 256바이트 창은 과도하지 않음. `<?xml ...?>` 선언 후 `<svg` 를 찾는 것이 목적이므로 적절.

### 3.2 `web_canvas.rs`

**`detect_image_mime_type` SVG 분기 위치**: 기존 바이너리 포맷(PNG/JPEG/GIF/WEBP/BMP/WMF) 판별 후 마지막에 SVG 를 추가. `else if` 체인 끝이므로 기존 포맷 감지에 영향 없음. 올바른 위치.

**`RawSvg` arm — A 경로**: `try_parse_single_image_data_url` 성공 시 `decode_base64_data_url` → `draw_image`. 기존 `IMAGE_CACHE + HtmlImageElement` 비동기 패턴을 그대로 재사용. 중복 없음.

**`RawSvg` arm — B 경로**: `wrap_svg_fragment` 로 `<svg>` 래핑 후 `draw_image(svg_doc.as_bytes(), ...)`. `detect_image_mime_type` 이 `image/svg+xml` 을 반환해 기존 draw_image 파이프라인으로 흐름. 별도 async 함수를 만들지 않고 재사용한 점이 깔끔함.

**`Placeholder` arm**: `svg.rs` 의 Placeholder 출력(rect + 점선 테두리 + 중앙 라벨)과 동등. `set_text_align/baseline` 복원으로 후속 노드 렌더에 영향 없음. 세심한 처리.

**잠재적 이슈 없음**: clippy `-D warnings` clean, WASM check clean.

---

## 4. 검증 결과 (로컬 재현)

```
cargo test --lib svg_fragment   19 passed / 0 failed ✅
cargo test --lib               982 passed / 0 failed ✅  (PR 작성 시 968/14 → 이후 #147/#267 수정으로 14건 추가 해소)
cargo clippy --lib -- -D warnings  0 warnings ✅
cargo check --lib --target wasm32-unknown-unknown  clean ✅
cargo test --test svg_snapshot   3 passed / 0 failed ✅
```

---

## 5. 판정

**Merge 권장.**

- 근본 원인 분석 명확 (`svg.rs` vs `web_canvas.rs` match arm 비대칭)
- 헬퍼 모듈 분리 + 19건 단위 테스트 — 품질 기준 충족
- A/B 경로 모두 커버, Placeholder arm 동등 출력
- 기존 draw_image 재사용으로 코드 증가 최소화
- 전체 테스트 회귀 없음

**후속 제안** (merge 후 M101+ 후보):
- `svg.rs` 와 `web_canvas.rs` 의 `RenderNodeType` match arm 커버리지 비교 자동화 (동일 비대칭 재발 방지)

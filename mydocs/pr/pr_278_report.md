# PR #278 최종 보고서 — Task #275: WASM canvas OLE RawSvg/Placeholder 처리 복구

**PR**: [#278](https://github.com/edwardkim/rhwp/pull/278)
**기여자**: @planet6897 (Jaeuk Ryu)
**처리 결정**: **Merge 권장**
**처리일**: 2026-04-24

---

## 1. 버그 요약

`samples/bitmap.hwp`, `samples/한셀OLE.hwp` 를 rhwp-studio 에서 열면 파일 로드는 성공하나 본문이 **완전히 빈 페이지**로 렌더됨.

- 원인: `web_canvas.rs` 의 `render_node` match 에서 `RenderNodeType::RawSvg` 와 `RenderNodeType::Placeholder` arm 이 누락 → `_ =>` 로 빠져 아무것도 렌더 안 됨
- `svg.rs` (CLI/네이티브)는 두 arm 처리 → CLI 는 정상, WASM 만 버그

---

## 2. 변경 내용

| 파일 | 변경 |
|------|------|
| `src/renderer/svg_fragment.rs` | 신규 (+279줄): SVG 조각 파서 유틸 5종 + 단위 테스트 19건 |
| `src/renderer/web_canvas.rs` | `Placeholder` arm (+26줄), `RawSvg` arm (+25줄), `detect_image_mime_type` SVG 분기 |

---

## 3. 검증 결과

```
cargo test --lib svg_fragment       19 passed / 0 failed ✅
cargo test --lib                   982 passed / 0 failed ✅
cargo clippy --lib -- -D warnings   0 warnings ✅
cargo check --lib --target wasm32   clean ✅
cargo test --test svg_snapshot       3 passed / 0 failed ✅
```

---

## 4. 판정 근거

- 근본 원인 분석 정확 — `svg.rs`↔`web_canvas.rs` match arm 비대칭
- 헬퍼 모듈 분리(`svg_fragment.rs`) + 19건 단위 테스트로 품질 보증
- `draw_image` 파이프라인 재사용 — 새 비동기 흐름 없이 기존 `IMAGE_CACHE` 패턴 공유
- `Placeholder` arm 의 `text_align/baseline` 복원으로 후속 노드 영향 없음
- 전체 테스트 회귀 없음

---

## 5. 후속 제안 (M101+)

`svg.rs` ↔ `web_canvas.rs` `RenderNodeType` match arm 커버리지 비교 자동화 (동일 비대칭 재발 방지)

---

## 6. 처리

- GitHub PR #278 Approve 완료
- @edwardkim 에게 Merge 요청

# Visual Diff 기반 HWP 호환성 확보 기획서

> 문서 하나씩 한컴 렌더링과 비교하여, 페이지 단위로 차이를 찾고 수정하는 체계

---

## 1. 목표

**rhwp의 렌더링 결과를 한컴 뷰어와 페이지 단위로 비교**하여, 시각적 차이를 정량화하고 문서별로 호환성을 달성한다.

### 성공 기준

| 지표 | 목표 |
|------|------|
| 페이지별 픽셀 일치율 (SSIM) | ≥ 95% |
| 문서별 페이지 수 일치 | 100% |
| 레이아웃 구조 일치 (문단/표 위치) | ≥ 98% |

### 기존 접근법과의 차이

| 기존 (T398~404) | Visual Diff |
|-----------------|-------------|
| LINE_SEG 필드 수치 비교 | 렌더링 결과 이미지 비교 |
| 글자 폭·줄바꿈 개별 추적 | 최종 출력물 기준 전체 품질 |
| 원인 분석 중심 | 결과 확인 → 원인 추적 |
| 제어 샘플 위주 | 실제 문서 위주 |

**두 접근법은 상호 보완적이다** — Visual Diff로 "어디가 다른지" 발견하고, LINE_SEG 비교로 "왜 다른지" 추적한다.

---

## 2. 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                     Visual Diff Pipeline                        │
│                                                                 │
│  ┌──────────┐   ┌──────────────┐   ┌───────────┐   ┌────────┐ │
│  │ 한컴 참조  │   │ rhwp 렌더링   │   │ 이미지 비교 │   │ 리포트 │ │
│  │ (Ground   │──▶│ (SVG→PNG)    │──▶│ (Diff)     │──▶│ (HTML) │ │
│  │  Truth)   │   │              │   │            │   │        │ │
│  └────���─────┘   └────────────���─┘   └────────���──┘   └────────┘ │
│       │                │                  │              │      │
│   PDF/PNG 캡처     export-svg         pixelmatch      per-page │
│   from 한컴        + resvg→PNG        + SSIM          diff map │
└─────────────────────────────────────────────────────────────────┘
```

### 2.1 Ground Truth 생성 (한컴 참조 이미지)

한컴에서 문서를 열어 **페이지별 PNG 이미지**를 확보한다.

| 방법 | 장점 | 단점 |
|------|------|------|
| **한컴 → PDF 인쇄 → pdf-to-png** | 자동화 가능, 해상도 제어 | PDF 변환 과정의 미세 차이 |
| **한컴 화면 캡처 (수동)** | 화면 표시 그대로 | 수동, 해상도 불균일 |
| **한컴 hwp2pdf CLI** | 완전 자동화 | 설치 필요, 라이선스 |

**권장**: 한컴에서 PDF 인쇄 → `pdftoppm` 또는 `pdf-lib` 로 페이지별 PNG 추출.
작업지시자가 한컴에서 PDF를 생성하고, 이후 파이프라인은 자동화한다.

### 2.2 rhwp 렌더링 이미지 생성

```bash
# SVG 내보내기 (페이지별)
rhwp export-svg sample.hwp -o output/rhwp/ --embed-fonts

# SVG → PNG 변환 (resvg, 이미 hwpx 프로젝트에서 사용 중)
resvg output/rhwp/page_0.svg output/rhwp/page_0.png --width 2480
```

**해상도 통일**: 양측 모두 A4 기준 **2480×3508px (300 DPI)** 로 정규화.

### 2.3 이미지 비교 엔진

```
per-page comparison:
  1. 크기 정규화 (resize to same dimensions)
  2. 픽셀 diff (pixelmatch — threshold 0.1)
  3. SSIM 계산 (structural similarity)
  4. 차이 영역 바운딩 박스 추출
  5. diff 히트맵 이미지 생성 (빨간색 = 차이)
```

**사용 도구**:
- `pixelmatch` (npm) — 고속 픽셀 비교, diff 이미지 생성
- `ssim.js` 또는 자체 SSIM — 구조적 유사도 (0.0~1.0)
- `sharp` (npm) — 이미지 리사이즈, 포맷 변환

### 2.4 리포트 생성

페이지별 비교 결과를 **HTML 리포트**로 자동 생성:

```
sample.hwp — Visual Diff Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Overall: 15 pages, SSIM avg 0.962, 12 pass / 3 fail

Page 1  [PASS]  SSIM: 0.991  diff: 0.2%
  ┌─────────┐ ��─────────┐ ┌─────────┐
  │ 한컴     │ │ rhwp    │ │ diff    │
  │ (참조)   │ │ (실제)   │ │ (차이)   │
  └─────────┘ └��────────┘ └────���────┘

Page 5  [FAIL]  SSIM: 0.871  diff: 8.3%
  ┌─────��───┐ ┌─────────┐ ┌─────────┐
  │ 한컴     │ │ rhwp    │ │ diff    │  ← 표 하단 20px 밀림
  └─��───────┘ └─────────┘ └─────────┘
  
  diff regions:
    - (120, 450) ~ (580, 520): 표 행 높이 차이
    - (50, 800) ~ (550, 830): 줄바꿈 위치 차이
```

---

## 3. 디렉토리 구조

```
visual-diff/
├── ground-truth/           # 한컴 참조 이미지 (작업지시자 제공)
│   ├── KTX/
│   │   ├── page_00.png
│   ��   ├── page_01.png
│   │   └── ...
│   ├── hwpspec/
│   │   └── ...
│   └── manifest.json       # 문서별 메타 (페이지 수, 해상도, 생성일)
│
├── rendered/               # rhwp 렌더링 결과 (자동 생성)
│   ├── KTX/
│   │   ├��─ page_00.png
│   │   └── ...
│   └── ...
│
├── diff/                   # 차이 이미지 + 리포트 (자동 생��)
│   ├── KTX/
│   │   ├── page_00_diff.png
│   │   └── ...
│   ├── KTX_report.html
│   └── summary.html        # 전체 문서 요약 대시보��
│
├── scripts/
│   ├── render-all.sh        # rhwp SVG→PNG 일괄 변환
│   ├── compare.mjs          # 이미지 비교 + diff 생성
│   ├── report.mjs           # HTML 리포트 생성
│   ├── gt-from-pdf.sh       # PDF→PNG ground truth 생성
│   └── run-pipeline.sh      # 전체 파이프라인 실행
│
├── config.json              # 설정 (threshold, resolution, 문서 목록)
└── README.md
```

---

## 4. 문서 우선순위 (문서별 순차 진행)

### Wave 1 — 기본 문서 (단순 구조, 빠른 검증)

| 순서 | 문서 | 페이지 | 특성 | 목적 |
|------|------|--------|------|------|
| 1 | `basic/KTX.hwp` | 1 | 표+이미지+텍스트 혼합 | 파이프라인 검증, 대표 문서 |
| 2 | `basic/english.hwp` | 1~2 | 영문 텍스트 | 영문 폭 측정 검증 |
| 3 | `basic/Textmail.hwp` | 1 | 단순 텍스트 | 기본 문단 레이아웃 |
| 4 | `basic/request.hwp` | 1~2 | 표 중심 | 표 렌더링 기본 |
| 5 | `basic/interview.hwp` | 1~2 | 텍스트+서식 | 글자 서식 |

### Wave 2 — 표/레이아웃 문서 (표, 다단, 이미지)

| 순서 | 문서 | 페이지 | 특성 | 목적 |
|------|------|--------|------|------|
| 6 | `hwp_table_test.hwp` | 2~3 | 다양한 표 | 표 셀 병합, 테두�� |
| 7 | `table-complex.hwp` | 1~2 | 복합 표 | 중첩 표, 셀 정렬 |
| 8 | `inner-table-01.hwp` | 1~2 | 중첩 표 | 표 안의 표 |
| 9 | `hwp-img-001.hwp` | 1~2 | 이미지 배치 | 그림 위치, 크기, 자르기 |
| 10 | `hwp-multi-001.hwp` | 3~5 | 다단 레이아웃 | 다단 분할 |

### Wave 3 — 복합 문서 (실무 수준)

| 순��� | 문서 | 페���지 | 특성 | 목적 |
|------|------|--------|------|------|
| 11 | `basic/treatise sample.hwp` | 5~10 | 논문 양식 | 머리말/꼬리말, 각주, 다단 |
| 12 | `footnote-01.hwp` | 2~3 | 각주 | 각주 영역 배치 |
| 13 | `endnote-01.hwp` | 2~3 | 미주 | 미주 처리 |
| 14 | `eq-01.hwp` | 1~2 | 수식 | 수식 렌더링 |
| 15 | `tac-case-001~005.hwp` | 각 1~2 | TAC 인라인 표 | 인라인 배치 |

### Wave 4 — 대형 문서 (종합 검증)

| 순서 | 문서 | 페이지 | 특성 | 목적 |
|------|------|--------|------|------|
| 16 | `hwpspec.hwp` | 170+ | HWP 스펙 문서 | 대량 페이지, 종합 |
| 17 | `hwpctl_API_v2.4.hwp` | 50+ | API 문서 | 표+코드+텍스트 혼합 |
| 18 | `basic/Worldcup_FIFA2010_32.hwp` | 30+ | 복합 레이아웃 | 이미지+표+서식 혼합 |

### Wave 5 — 특수 케이스

| 순서 | 문서 | 특성 | 목적 |
|------|------|------|------|
| 19 | `shift-return.hwp` | 강제 줄바꿈 | 줄바꿈 종류 |
| 20 | `form-01.hwp` | 양식 | 양식 필드 렌더링 |
| 21 | `draw-group.hwp` | 그리기 개체 | 도형 그룹 |
| 22 | `hwp-3.0-HWPML.hwp` | 구버전 포맷 | 하위 호환성 |

---

## 5. 문서별 진행 프로세스

**한 문서를 완전히 맞춘 후 다음 문서로 넘어간다.**

```
┌──────────────────────────────────────────────────────┐
│  Document-by-Document Conformance Cycle               │
│                                                       │
│  ① Ground Truth 확보                                  │
│     작업지시자: 한컴에서 PDF 인쇄 → 제공               │
│                                                       │
│  ② 파이프라인 실행                                     │
│     rhwp export-svg → resvg PNG → pixelmatch diff     │
│                                                       │
│  ③ 리포트 확인                                        │
│     페이지별 SSIM, diff 히트맵, 차이 영역 확인         │
│                                                       │
│  ④ 차이 분석                                          │
│     diff 영역 → dump-pages / dump / debug-overlay     │
│     로 원인 특정 (줄바꿈? 표 높이? 이미지 위치?)       │
│                                                       │
│  ⑤ 수정                                              │
│     원인별 코드 수정 (렌더러, 조판, 파서)              │
│                                                       │
│  ⑥ 재검증                                             │
│     파이프라인 재실행 → SSIM 개선 확인                  │
│     + 이전 문서 회귀 확인 (regression guard)           │
│                                                       │
│  ⑦ 완료 판정                                          │
│     모든 페이지 SSIM ≥ 0.95 → PASS → 다음 문서        │
│                                                       │
│  ※ 수정 불가 케이스 (폰트 차이 등)는 known-diff로     │
│    등록하고 해당 영역을 마스크 처리                     │
└─────────────────────────────────��────────────────────┘
```

### 회귀 방지 (Regression Guard)

- 새 문서 작업 시 **이전에 통과한 모든 문서**도 함께 비교
- 회귀 발생 시: 새 수정 롤백 또는 조건부 분기 도입
- `summary.html` 대시보드에서 전체 문서 상태 한눈에 확인

```
Visual Diff Dashboard
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Document              Pages  SSIM   Status
  ─────────────────────────────────────
  KTX.hwp                  1   0.98   ✅ PASS
  english.hwp              2   0.96   ✅ PASS
  Textmail.hwp             1   0.97   ✅ PASS
  request.hwp              2   0.93   🔧 WIP (표 테두리)
  hwp_table_test.hwp       3   --     ⏳ PENDING
  ...
```

---

## 6. Known-Diff 마스크 정책

일부 차이는 rhwp에서 의도적으로 다르거나, 현 단계에서 해결 불가능하다.

| 유형 | 예시 | 처리 |
|------|------|------|
| **폰트 차이** | 한컴 전용 폰트 → 오픈소스 대체 | 텍스트 영역 마스크, 레이아웃만 비교 |
| **안티앨리어싱** | SVG vs 한컴 GDI 렌더링 차이 | threshold 완화 (0.1→0.15) |
| **서브픽셀** | 1px 미만 위치 차이 | SSIM 임계값으로 흡수 |
| **커서/선택** | 한컴 캡처 시 커서 포함 | 캡처 시 커서 숨기기 |

마스크 파일 (`mask.json`):
```json
{
  "KTX": {
    "page_00": [
      {
        "region": [100, 200, 400, 50],
        "reason": "한컴 전용 폰트(HY견고딕) → 오픈소스 대체",
        "type": "font-substitution"
      }
    ]
  }
}
```

---

## 7. 기술 구현 상세

### 7.1 Ground Truth 생성 스크립트

```bash
# gt-from-pdf.sh — PDF → 페이지별 PNG
# 의존성: poppler-utils (pdftoppm)
pdftoppm -png -r 300 "$PDF_FILE" "$OUTPUT_DIR/page"
# 결과: page-01.png, page-02.png, ...
# 파일명 정규화: page_00.png, page_01.png, ... (0-indexed)
```

### 7.2 rhwp 렌더링 스크립트

```bash
# render-all.sh — 문서별 SVG→PNG 일괄 변환
for doc in $(jq -r '.documents[].file' config.json); do
    name=$(basename "$doc" .hwp)
    mkdir -p rendered/"$name"
    
    # SVG 내보내기 (임베드 폰트)
    rhwp export-svg "samples/$doc" -o rendered/"$name" --embed-fonts
    
    # SVG → PNG (300 DPI, A4 기준)
    for svg in rendered/"$name"/*.svg; do
        png="${svg%.svg}.png"
        resvg "$svg" "$png" --width 2480
    done
done
```

### 7.3 비교 엔진 핵심 로직

```javascript
// compare.mjs (Node.js)
import { PNG } from 'pngjs';
import pixelmatch from 'pixelmatch';

function comparePage(gtPath, renderedPath, diffPath, options) {
    const gt = PNG.sync.read(fs.readFileSync(gtPath));
    const rendered = PNG.sync.read(fs.readFileSync(renderedPath));
    
    // 크기 정규화
    const [w, h] = [Math.max(gt.width, rendered.width),
                     Math.max(gt.height, rendered.height)];
    const gtResized = resizeToFit(gt, w, h);
    const renderedResized = resizeToFit(rendered, w, h);
    
    // 마스크 적용
    const mask = loadMask(options.document, options.page);
    if (mask) applyMask(gtResized, renderedResized, mask);
    
    // 픽셀 비교
    const diff = new PNG({ width: w, height: h });
    const mismatchCount = pixelmatch(
        gtResized.data, renderedResized.data, diff.data,
        w, h, { threshold: 0.1 }
    );
    
    // SSIM 계산
    const ssim = calculateSSIM(gtResized, renderedResized);
    
    // diff 이미지 저장
    fs.writeFileSync(diffPath, PNG.sync.write(diff));
    
    return {
        mismatchPixels: mismatchCount,
        mismatchRate: mismatchCount / (w * h),
        ssim: ssim,
        pass: ssim >= 0.95,
    };
}
```

### 7.4 config.json 예시

```json
{
    "resolution": { "dpi": 300, "width": 2480, "height": 3508 },
    "threshold": { "pixel": 0.1, "ssim_pass": 0.95 },
    "documents": [
        {
            "file": "basic/KTX.hwp",
            "name": "KTX",
            "pages": 1,
            "wave": 1,
            "status": "pending"
        },
        {
            "file": "basic/english.hwp",
            "name": "english",
            "pages": 2,
            "wave": 1,
            "status": "pending"
        }
    ]
}
```

---

## 8. 기존 인프라와의 연계

| 기존 도구 | Visual Diff에서의 역할 |
|----------|----------------------|
| `export-svg` | rhwp 렌더링 이미지 생성의 핵심 |
| `--debug-overlay` | diff 영역의 원인 분석 시 문단/표 식별 |
| `dump-pages` | diff 영역 → 해당 페이지의 문단 배치 확인 |
| `dump -s N -p M` | 특정 문단의 ParaShape, LINE_SEG 상세 조사 |
| `ir-diff` | HWPX↔HWP 포맷 간 IR 차이 확인 (보조) |
| T398 LINE_SEG 비교 | 줄바꿈 차이의 수치적 원인 추적 |
| E2E scenario-runner | 편집 후 레이아웃 안정성 검증 (보조) |

### 디버깅 워크플로우 통합

```
① Visual Diff → Page 5 FAIL (SSIM 0.87)
② diff 히트맵 → (120, 450)~(580, 520) 영역에 차이
③ debug-overlay → 해당 영역은 s0:pi=45 ci=0 (표 16x4)
④ dump-pages -p 5 → pi=45 표 높이 확인
⑤ dump -s 0 -p 45 → ParaShape, LINE_SEG 상세
⑥ LINE_SEG 비교 (T398) → text_start -3 (글자 폭 차이)
⑦ 수정 → 재검증
```

---

## 9. 구현 단계

### Step 1: 파이프라인 기반 구축

- `visual-diff/` 디렉토리 생성
- `compare.mjs` — pixelmatch + SSIM 비교 엔진
- `report.mjs` — HTML 리포트 생성기
- `render-all.sh` — rhwp SVG→PNG 일괄 변환
- `gt-from-pdf.sh` — PDF→PNG ground truth 생성
- `config.json` — 설정 + 문서 목록
- `run-pipeline.sh` — 전체 파이프라인 원클릭 실행

**산출물**: `KTX.hwp` 1개 문서로 파이프라인 end-to-end 동작 확인

### Step 2: KTX.hwp (Wave 1-1) — 첫 번째 문서 맞추기

- 작업지시자가 한컴에서 KTX.hwp PDF 생성 → ground-truth 이미지 확보
- 파이프라인 실행 → 첫 리포트 생성
- diff 분석 → 수정 → SSIM ≥ 0.95 달성
- known-diff 마스크 정책 실전 적용

### Step 3: Wave 1 나머지 (english, Textmail, request, interview)

- 문서별 순차 진행
- 각 문서 PASS 후 이전 문서 regression guard 실행
- 반복적으로 발견되는 패턴은 **공통 수정**으로 분류

### Step 4: Wave 2~5 순차 진행

- Wave별 복잡도 증가에 따라 수정 범위도 확대
- Wave 4 (대형 문서) 진입 시점에서 SSIM 기준 재조정 검토
- 최종 summary dashboard 완성

### Step 5: CI 통합 (선택)

- GitHub Actions에서 regression guard 자동 실행
- PR 머지 전 기존 PASS 문서 회귀 확인
- ground-truth 이미지는 Git LFS 또는 별도 스토리지

---

## 10. 작업 분담

| 역할 | 담당 | 내용 |
|------|------|------|
| **Ground Truth 생성** | 작업지시자 | 한컴에서 PDF 인쇄, 제공 |
| **파이프라인 구축** | Claude (이 태스크) | 스크립트, 비교 엔진, 리포트 |
| **diff 분석** | 공동 | 리포트 보면서 원인 토론 |
| **코드 수정** | rhwp 레포에서 진행 | 렌더러/조판/파서 수정 |
| **Known-diff 판정** | 작업지시자 | 수용/거부 결정 |

---

## 11. 의존성

| 도구 | 용도 | 설치 |
|------|------|------|
| `resvg` | SVG→PNG 변환 | `cargo install resvg` 또는 `brew install resvg` |
| `pdftoppm` (poppler) | PDF→PNG | `brew install poppler` |
| `pixelmatch` | 픽셀 비교 | `npm install pixelmatch pngjs` |
| `sharp` | 이미지 리사이즈 | `npm install sharp` |
| rhwp (native build) | SVG 내보내기 | 기존 빌드 |

---

## 12. 리스크 및 대응

| 리스크 | 영향 | 대응 |
|--------|------|------|
| 한컴 폰트 없음 → 텍스트 영역 대량 차이 | SSIM 급락 | 텍스트 영역 마스크 + 레이아웃만 비교 |
| 대형 문서 (170+ 페이지) 처리 시간 | 파이프라인 느림 | 변경 페이지만 증분 비교 |
| SVG→PNG 변환 품질 (resvg 한계) | 오탐 (false diff) | threshold 조정 + resvg 최신 버전 |
| 한컴 PDF 출력과 화면 출력의 미세 차이 | 기준 흔들림 | 한컴 화면 캡처로 크로스 체크 |
| 수정이 다른 문서에 regression 유발 | 무한 루프 | regression guard 강제 + 조건부 분기 |

---

## 승인 요청

위 기획서를 검토하시고, 다음을 결정해 주세요:

1. **진행 승인** — Step 1 (파이프라인 기반 구축)부터 시작
2. **Ground Truth 첫 문서** — `basic/KTX.hwp`를 한컴에서 PDF로 제공 가능 여부
3. **문서 우선순위 조정** — Wave 1~5 순서 변경이 필요한 경우
4. **SSIM 기준** — 0.95가 적절한지, 조정 필요 여부

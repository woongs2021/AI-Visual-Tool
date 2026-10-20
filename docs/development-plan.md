# AI Visual Cue 제작 툴 — 개발 계획서

> 버전 v0.1 · 2026-10-20 · 작성: 웅 / Claude
> 범위: **계획 수립만** (구현 없음)

---

## 목차

0. [불명확해서 가정한 사항](#0-불명확해서-가정한-사항)
1. [레퍼런스 분석 — Reference video](#1-레퍼런스-분석--reference-video)
2. [제품 개요와 핵심 설계 원칙](#2-제품-개요와-핵심-설계-원칙)
3. [기술 스택 비교 및 결정](#3-기술-스택-비교-및-결정)
4. [단일 상태(State) 설계 — `CueSpec`](#4-단일-상태state-설계--cuespec)
5. [비주얼 큐 사양 상세 (형태 · 레벨 · 타임라인 · 물성)](#5-비주얼-큐-사양-상세)
6. [Phase별 계획](#6-phase별-계획)
7. [참조 자료 × Phase 매핑](#7-참조-자료--phase-매핑)
8. [리스크와 대응](#8-리스크와-대응)
9. [범위 밖 / 추후 과제](#9-범위-밖--추후-과제)
10. [부록 — 폴더 구조, 코드 생성 출력 예시 골격](#10-부록)

---

## 0. 불명확해서 가정한 사항

요구사항 중 해석이 갈리는 부분은 아래처럼 가정하고 진행한다. 각 항목은 Phase 0 종료 시점에 확정(✅) 또는 수정한다.

### 0-1. 첨부 자료 관련

| # | 가정 | 근거 / 영향 |
|---|---|---|
| A1 | 이번 계획서에서는 **Reference video(인스타그램 화면 녹화, field_io "Quantum/Qubit" 포스트)만 참고**한다. Reference effect / Reference level / App examples 이미지는 **반영하지 않았다.** | 요청자 지시. 해당 이미지가 필요한 결정(물성 타입명, 레벨 레이아웃, 앱 UI 목업)은 이 문서의 가정으로 대신하고, 이미지가 제공되면 Phase 0의 "레퍼런스 대조" 작업에서 교체한다. |
| A2 | Reference video에서 가져오는 것은 **①루프 리듬(유지→빠른 전환→안착), ②이징 성격, ③그라데이션 구의 셰이딩 방식, ④3스톱 팔레트와 스톱 위치 표현 UI**다. 영상의 "팔레트 순환" 자체는 우리 제품의 **레벨 순환(L1→L2→L3→L1)**에 대응시킨다. | 프롬프트의 "모션 타이밍/루프 방식은 Reference video를 따를 것" |
| A3 | 영상의 전환(구가 회전하며 색면이 빠지고 다음 색이 반대편에서 들어오는 '위상 전환')은 **레벨 전환 시 '물성 유지 + 형태 변형'으로 재해석**한다. 위상 회전 효과 자체는 옵션 토글(`transitionStyle: "morph" \| "phase"`)로 제공한다. | 프롬프트는 "기본 물성이 유지된 채 자연스럽게 퍼져서 안착"이 중요하다고 했으므로 기본값은 morph. |

### 0-2. 제품 / 화면 관련

| # | 가정 |
|---|---|
| P1 | 툴 타이틀은 임시로 **"AI Visual Cue Studio"**로 한다. |
| P2 | `About` 탭은 정적 페이지다. 툴 소개, 사용법, 버전, 생성 코드의 최소 요구사항(minSdk 등)을 담는다. |
| P3 | 다크/라이트 토글은 **툴 UI 테마**를 바꾼다. 프리뷰 캔버스 안의 "앱 배경"은 별도 값(`preview.appTheme`)이며, 기본값은 툴 테마를 따른다(=`"auto"`). |
| P4 | 테마 초기값은 OS 설정(`prefers-color-scheme`)이고, 사용자가 바꾼 값은 `localStorage`에 저장한다. |
| P5 | 좌측 컨트롤 패널은 **최소 360px, 최대 480px, 기본 1/3**이다. 1024px 미만 화면에서는 패널을 하단 시트로 전환한다(모바일은 보조 지원). |
| P6 | 프리뷰 캔버스 기준은 **360×780dp (1080×2340px @3x)**이고, 영역 높이에 맞춰 **비율을 고정한 채 축소**한다. 1번 뷰(두 영상 나란히)는 두 캔버스를 같은 배율로 나란히 둔다. |
| P7 | 프리뷰 3뷰 전환 방법: 좌우 화살표 버튼, 키보드 ←/→, 포인터 드래그/스와이프(임계값 20% 또는 속도 0.5px/ms), 하단 페이지 인디케이터(●○○). |
| P8 | "무한 루프"는 세 뷰가 **같은 타임라인 시계(clock)를 공유**한다는 뜻으로 해석한다. 1번 뷰의 영상①·②가 완전히 동기화되어 재생된다. |
| P9 | 프리뷰에는 **재생/일시정지, 0.25×–2× 속도, 타임라인 스크러버, 레벨 점프(L1/L2/L3)** 같은 보조 컨트롤을 둔다. 요구사항에는 없지만 디자이너가 모션을 검토하는 데 필요하다고 판단했다. |
| P10 | Code 탭은 **읽기 전용**이다. 코드를 수정해서 프리뷰에 역반영(round-trip)하는 기능은 범위 밖이다. 대신 **설정 JSON 내보내기/불러오기**를 제공한다. |

### 0-3. 비주얼 큐 사양 관련

| # | 가정 |
|---|---|
| V1 | "화면에 납작하게 붙어 있는 상태"는 **화면 상단 중앙의 얇은 알약(pill) 형태**(지름의 약 60% 폭, 4dp 두께, 그림자 없음)로 정의한다. Level 1의 시작 형태다. |
| V2 | "구로 떠올라 부유"는 3D 셰이딩된 원형(실제 3D 메시가 아닌 2.5D 셰이더 표현) + 하단 드롭섀도 + ±3dp의 느린 상하 부유(floating bob)로 표현한다. |
| V3 | "이동 후 다시 안착"은 구가 목표 위치로 이동하면서 **둥근 사각형(카드)으로 변형되어 납작해지는 것**이다. 이때 그림자가 줄어들며 평면에 붙는 느낌을 준다. |
| V4 | "새 컴포넌트(리스트 등) 생성"은 안착한 카드 내부/하단에 **리스트 아이템 3개가 60ms 간격(stagger)으로 페이드+슬라이드업**되는 것으로 정의한다. 영상① 단독 뷰에서는 텍스트 없는 플레이스홀더 바(skeleton)로, 영상② 앱 뷰에서는 목업 리스트 셀로 보여준다. |
| V5 | **Level 1** = 상단 중앙에서 구 생성 및 부유. **Level 2** = 구가 화면 중하단으로 넓게 퍼지며 리스트 카드(폭 = 화면 폭 − 32dp)로 안착. **Level 3** = 카드가 전체 화면으로 확산(모서리 반경 0, 엣지 글로우만 남음). **L3→L1**은 전체 화면이 상단으로 수축하며 다시 구가 되는 것이다. |
| V6 | "안착 시 높이/두께" = 안착 카드의 **높이(dp)**와, 카드 테두리 쪽 물성 볼륨감을 정하는 **두께(bevel, dp)**로 해석한다. |
| V7 | 그라데이션 스톱은 **최소 1개, 최대 3개**(1개면 단색)다. 각 스톱은 `color(hex)`, `position(0–1)`, `name(선택)`을 갖는다. 영상의 컬러칩 세로선 UI를 그대로 차용하며, **칩의 세로 위치 = 스톱 위치**다. |
| V8 | 그라데이션 기본 매핑은 영상과 같은 **"상단 내부 하이라이트 → 가장자리/하단 밝은 색" 방사형**이다. `gradientType: radial \| linear \| sweep` 옵션을 둔다. |
| V9 | 물성 타입은 Reference effect 이미지를 반영하지 않고(A1), **5종(Soft Glow / Frosted Glass / Liquid Metal / Pearl / Matte Clay)**으로 임시 정의한다(§5-4). |
| V10 | 출력 코드 대상은 **Android 13(API 33)+**에서 AGSL RuntimeShader를 쓰는 고품질 경로다. **API 26–32**에서는 Brush 그라데이션과 그림자로 근사하는 폴백 경로를 함께 생성한다. 생성 코드의 `minSdk`는 26이다. |
| V11 | 생성 코드는 **단일 Kotlin 파일**로 복사·붙여넣기만 하면 동작해야 한다. 외부 의존성은 Compose BOM 기본 라이브러리(ui, foundation, animation, material3)만 사용한다. |

### 0-4. 일정 / 팀

| # | 가정 |
|---|---|
| T1 | 개발자 1인과 디자이너 리뷰어 1인을 기준으로 하며, 총 **약 6–7주**로 추정한다(§6 각 Phase에 예상 기간 기재). |
| T2 | 산출물은 정적 호스팅(Vercel/Netlify/GitHub Pages)이 가능한 SPA다. 백엔드는 없다. |

---

## 1. 레퍼런스 분석 — Reference video

> 원본: `Screen_Recording_20261019_182649_Instagram.mp4` (1080×2520, 16.5s, 화면 녹화)
> 분석 방법: 10fps로 프레임을 추출하고 구 영역의 채색 비율과 중심 픽셀 색을 시계열로 측정했다.

### 1-1. 화면 구성

- **상단 영역**: 밝은 회색 배경(#F5F5F5 계열). 좌상단에 "Quantum", 우상단에 "Qubit" 라벨(작은 회색 산세리프). 중앙에 그라데이션 구가 있다(화면 폭 대비 지름 약 45%).
- **하단 영역**: 같은 팔레트의 거대한 방사형 그라데이션 원호가 화면을 채운다. 중앙에는 **세로선으로 연결된 컬러칩 3개**가 있다(□ + 색 이름 + HEX). 좌하단에 페이지 번호(`01 / 03`), 우하단에 팔레트 이름이 있다.

### 1-2. 팔레트 (실측)

| # | 팔레트 이름 | Stop 1 | Stop 2 | Stop 3 | 스톱 위치(칩 세로 위치 기준 추정) |
|---|---|---|---|---|---|
| 01 | (가려져 확인 불가, "T…") | Indigo `#5439D6` | Azure `#3873F0` | Bloom `#E8C9FF` | 0.00 / 0.62 / 1.00 |
| 02 | Azure | Iris `#6449F5` | Azure `#3A77FF` | Mist `#C9EEFF` | 0.00 / 0.40 / 1.00 |
| 03 | Blossom | Lilac `#7B72EE` | Orchid `#E68FF0` | Blush `#FFDCE9` | 0.00 / 0.80 / 1.00 |

→ **이 3개를 툴의 기본 프리셋으로 채택**한다. 기본 선택은 02 Azure.
→ **스톱 위치가 팔레트마다 다르고, 칩의 세로 위치가 그 위치를 나타낸다.** 이 방식을 컨트롤 패널의 그라데이션 에디터 UI로 그대로 차용한다(V7).

### 1-3. 구 셰이딩

- 중심보다 약간 위(지름의 약 30% 지점)에 **Stop 1(가장 진한 보라/남색) 하이라이트 코어**가 있다.
- 바깥으로 갈수록 Stop 2를 거쳐 가장자리와 하단에서 **Stop 3(아주 밝은 색)**이 된다. 하단 림(rim)이 특히 밝아 **역광/서브서피스 느낌**이 난다.
- 외곽선이 없고, 가장자리가 1–2px 정도 부드럽게 안티앨리어싱되어 있다. 드롭섀도는 없다(우리 제품은 부유 시 그림자를 추가한다).
- 유지 구간에서도 **하이라이트 코어가 천천히 이동**한다(중심 픽셀 색이 2초간 서서히 변화). 살아 있는 느낌을 주는 미세 모션이다.

### 1-4. 루프 타이밍 (실측)

전환 시점(구 채색 비율이 최저가 되는 순간): **2.5s → 5.9s → 9.4s → 12.8s → 16.2s**

| 항목 | 값 | 비고 |
|---|---|---|
| 한 상태의 주기 | **≈ 3.43s** | 3개 상태 → **전체 루프 ≈ 10.3s** |
| 유지(hold) | ≈ 2.6s | 하이라이트 미세 드리프트 진행 |
| 전환(transition) 총 길이 | ≈ 0.8s | 아래 3구간으로 나뉨 |
| ├ 빠지기(out) | ≈ 0.3s | **ease-in(가속)**. 색면이 회전하며 빠짐 |
| ├ 공백(blank) | ≈ 0.1s | 무채색 구(배경과 거의 같은 #EDEDED) |
| └ 들어오기(in) | ≈ 0.4s | **ease-out(감속)**. 반대편 대각선 경계로 진입 |
| 안착 후 셰이딩 정착(settle) | ≈ 0.5–0.6s | 하이라이트가 제자리로 감속 이동 |
| 하단 패널 선행 | ≈ 0.6s | 하단 그라데이션은 구보다 약 0.6s 먼저 크로스페이드를 시작한다(**계층적 시차**) |

→ **우리 기본값**: 레벨당 `hold 2600ms + transition 800ms`(out 300 / blank 100 / in 400, morph 모드에서는 blank를 0으로 하고 out+in을 연속 morph 800ms로 처리), settle 550ms, 주변 UI 시차 ±150–600ms.
→ 이징 근사: out = `cubic-bezier(0.55, 0, 1, 0.45)`, in = `cubic-bezier(0.05, 0.7, 0.1, 1)`(Material emphasized-decelerate 계열), settle = `cubic-bezier(0.2, 0, 0, 1)`. 웹(CSS/JS)과 Compose `CubicBezierEasing`에 **같은 4개 숫자**를 사용한다.

---

## 2. 제품 개요와 핵심 설계 원칙

### 2-1. 한 줄 정의

AI 에이전트 활성화를 알리는 비주얼 큐의 **물성·컬러·크기·모션**을 디자이너가 웹에서 조작하고, 그 결과를 **그대로 동작하는 Kotlin + Jetpack Compose 코드**로 내보내는 디자인 툴.

### 2-2. 핵심 설계 원칙

1. **Single Source of Truth**: 모든 컨트롤 값은 하나의 직렬화 가능한 객체 `CueSpec`에 담는다. 프리뷰 렌더러와 코드 생성기는 둘 다 `CueSpec`을 입력으로 받는 **순수 함수/소비자**다.
2. **셰이더 동형성(Shader Parity)**: 웹 프리뷰의 프래그먼트 셰이더와 Android의 AGSL 셰이더를 **하나의 셰이더 템플릿에서 생성**한다. 웹에서 보이는 것과 기기에서 보이는 것의 차이를 구조적으로 최소화하기 위해서다.
3. **타임라인 동형성**: 모션은 "키프레임 테이블 + 이징 4숫자" 같은 데이터로 정의한다. 웹은 이 테이블을 JS 타임라인 엔진으로, Compose는 생성된 `keyframes {}` / `Animatable` 코드로 재생한다. 하드코딩 상수는 없다.
4. **결정적(deterministic) 렌더링**: 프리뷰는 `t`(ms)만 주면 어떤 프레임이든 같은 결과를 그린다. 스크러빙, 일시정지, 스크린샷 테스트가 가능해진다.

```
            ┌──────────────── CueSpec (Zustand store, JSON) ────────────────┐
            │  colors · material · size · motion · levels · preview opts      │
            └───────┬───────────────────────────┬───────────────────────┬────┘
                    │                           │                       │
          derive (selectors)            derive (pure fn)          serialize
                    │                           │                       │
   ┌────────────────▼──────────┐   ┌────────────▼─────────────┐   ┌─────▼──────┐
   │ Timeline engine (t → frame│   │ Code generator           │   │ JSON 내보내기│
   │ params) + WebGL2 renderer │   │  • AGSL 셰이더 문자열     │   │ /불러오기   │
   │  ← shader template(GLSL)  │   │  • Kotlin/Compose 파일    │   └────────────┘
   └───────────────────────────┘   │  ← 같은 shader template   │
                                   └──────────────────────────┘
```

---

## 3. 기술 스택 비교 및 결정

### 3-1. 프리뷰 렌더링 방식 비교

| 기준 | CSS 애니메이션 (+ SVG 필터) | Canvas 2D | **WebGL2 (프래그먼트 셰이더)** |
|---|---|---|---|
| 그라데이션 구 셰이딩 (영상 수준) | △ radial-gradient 중첩으로 근사, 3스톱 + 하이라이트 이동 시 부자연스러움 | △ createRadialGradient로 근사, 노멀 기반 셰이딩 불가 | ◎ 노멀 재구성 → 정확한 볼륨감 |
| 물성(글래스 굴절, 메탈 반사, 프레넬, 노이즈) | ✕ backdrop-filter 정도 | ✕ 픽셀 단위 루프는 성능 불가 | ◎ |
| 구 ↔ 둥근 사각형 ↔ 전체 화면 morph | △ border-radius 보간 가능하나 셰이딩이 형태를 못 따라감 | △ | ◎ SDF로 연속 morph |
| Android 출력과의 동형성 | ✕ Compose와 구조가 다름 | △ Brush API와 유사 | ◎ **GLSL ↔ AGSL 문법이 거의 1:1** |
| 성능 (60fps, 2개 캔버스 동시) | ◎ | ○ | ◎ (GPU) |
| 구현 난이도 | 낮음 | 중간 | 중간–높음 |
| 폴백 | — | — | WebGL2 미지원 시 Canvas2D 근사(희귀, 최신 브라우저 모두 지원) |

**결정: WebGL2 + 직접 작성한 프래그먼트 셰이더 (단일 풀스크린 쿼드).**
- three.js 같은 3D 엔진은 쓰지 않는다. 메시 기반 3D는 AGSL로 옮길 수 없어 동형성이 깨지기 때문이다. 렌더링은 **2D SDF + 2.5D 셰이딩**만으로 한다.
- 소형 래퍼로 `twgl.js`(~12KB)를 쓰거나 직접 구현한다. Phase 0에서 확정한다.
- **UI 레이어(리스트 아이템, 목업 앱 UI, 텍스트)는 DOM/CSS로** 셰이더 캔버스 위에 겹친다. Compose에서 셰이더 박스 위에 일반 Composable을 겹치는 구조와 맞추기 위해서다.

### 3-2. 앱(툴) 프레임워크

| 항목 | 선택 | 이유 |
|---|---|---|
| 언어 | TypeScript | `CueSpec` 스키마 타입 안정성 |
| UI | React 18 | 요구사항에 명시, 패널 UI 컴포넌트화 |
| 빌드 | Vite | 빠른 HMR, `?raw`로 셰이더 파일 임포트 |
| 상태 | Zustand + `immer` + `zundo`(undo/redo) | 단일 스토어, 셀렉터 기반 파생, 히스토리 |
| 스키마 검증 | Zod | JSON 불러오기 검증, 기본값 주입 |
| 스타일 | CSS Modules + CSS 변수(디자인 토큰) | 다크/라이트 토큰 전환 단순 |
| 컬러 피커 | `react-colorful`(2.8KB) + HEX 입력 | 경량 |
| 코드 하이라이트 | Shiki(Kotlin grammar, 빌드 시 로드) | 정확한 Kotlin 하이라이트, 테마 2종 |
| 제스처 | Pointer Events 직접 구현(또는 `@use-gesture/react`) | 3뷰 스와이프 |
| 테스트 | Vitest(스키마/생성기 단위), Playwright(E2E + 스크린샷) | |
| Android 검증 | 별도 `android-harness/` Gradle 프로젝트 + GitHub Actions | 생성 코드의 컴파일 검증 |

### 3-3. Android 출력 경로 결정

| 경로 | API | 사용처 |
|---|---|---|
| **AGSL `RuntimeShader` + `ShaderBrush`** (`drawWithCache` 안에서 `drawRect(ShaderBrush(shader))`) | 33+ | 기본 고품질 경로. 셰이딩, 물성, morph 전부 |
| `RenderEffect.createRuntimeShaderEffect` (`graphicsLayer { renderEffect = … }`) | 33+ | Frosted Glass 타입처럼 **배경을 샘플링**해야 하는 물성에 사용(`uniform shader content`) |
| `Brush.radialGradient` + `Modifier.shadow` + `RoundedCornerShape` 보간 | 26–32 | 폴백. 형태, 그라데이션, 모션은 유지하고 물성은 Soft Glow 근사로 낮춤 |
| `Modifier.blur` | 31+ | 폴백 중 Glow 번짐(31–32만) |

> 제안하지 않는 것: OpenGL ES `GLSurfaceView` 임베드(Compose 통합과 유지보수 비용이 큼), Lottie/동영상 렌더(파라미터 동기화 불가).

---

## 4. 단일 상태(State) 설계 — `CueSpec`

### 4-1. 스키마 (TypeScript, 요약)

```ts
type Hex = `#${string}`;

interface ColorStop { id: string; color: Hex; position: number; name?: string } // position 0..1

interface CueSpec {
  version: 1;
  colors: {
    stops: ColorStop[];             // length 1..3
    gradientType: 'radial' | 'linear' | 'sweep';
    angleDeg: number;               // linear/sweep 기준각
    highlightOffset: { x: number; y: number }; // 하이라이트 코어 위치(-1..1), 기본 {0,-0.3}
    presetId?: 'azure' | 'blossom' | 'indigo-bloom' | null;
  };
  material: {
    type: 'softGlow' | 'frostedGlass' | 'liquidMetal' | 'pearl' | 'matteClay';
    params: Record<string, number>; // 타입별 파라미터(§5-4), Zod로 타입별 스키마 분기
  };
  size: {
    sphereDiameterDp: number;       // 기본 160, 범위 48–320
    flatWidthRatio: number;         // 납작 상태 폭/지름, 기본 0.6
    flatThicknessDp: number;        // 기본 4
    settledHeightDp: number;        // 안착 카드 높이, 기본 220
    settledThicknessDp: number;     // 안착 bevel 두께, 기본 6
    settledCornerDp: number;        // 기본 28
    marginDp: number;               // 화면 가장자리 여백, 기본 16
  };
  shadow: { intensity: number; blurDp: number; offsetYDp: number }; // 부유 시 최대값
  motion: {
    speed: number;                  // 전체 배속 0.25–2, 기본 1
    floatAmplitudeDp: number;       // 기본 3
    floatPeriodMs: number;          // 기본 2400
    highlightDrift: number;         // 0–1, 유지 중 하이라이트 이동량
    transitionStyle: 'morph' | 'phase';
    easing: { out: Bezier; in: Bezier; settle: Bezier };
    levels: { L1: LevelTiming; L2: LevelTiming; L3: LevelTiming };
    componentStaggerMs: number;     // 기본 60
  };
  preview: {
    appTheme: 'auto' | 'light' | 'dark';
    showGuides: boolean;            // dp 그리드, 안전영역
  };
}
type Bezier = [number, number, number, number];
interface LevelTiming { enterMs: number; holdMs: number } // 기본 enter 800, hold 2600
```

### 4-2. 파생 규칙

- `deriveTimeline(spec) → Keyframe[]`: 레벨별 enter/hold와 speed를 반영해 전체 루프 길이와 키프레임을 계산한다.
- `sampleFrame(spec, timeline, tMs) → FrameParams`: 특정 시각의 `{center, halfSize, cornerRadius, elevation, morph, phaseAngle, shadow, highlight, componentProgress[]}`를 계산한다. **웹 렌더러는 이 값을 uniform으로 넘기고, Compose 생성기는 같은 키프레임을 Kotlin 코드로 출력**한다.
- `generateCompose(spec) → { kotlin: string, agsl: string, warnings: string[] }`: **순수 함수**다. 같은 입력이면 같은 출력(스냅샷 테스트 가능)을 낸다.

### 4-3. 상태 영속성

- URL 해시에 압축 JSON(`lz-string`)을 넣어 **링크 공유**를 지원하고, 마지막 상태는 `localStorage`에 자동 저장한다.
- JSON 파일 내보내기/불러오기를 지원하며, Zod 검증 실패 시 필드 단위 에러를 보여준다.

---

## 5. 비주얼 큐 사양 상세

### 5-1. 기하(geometry) 모델 — 하나의 SDF로 모든 형태 표현

모든 상태를 **둥근 사각형 SDF** 하나로 표현하고 파라미터만 보간한다.

| 상태 | halfSize (w,h) | cornerRadius | elevation(볼륨) | 그림자 |
|---|---|---|---|---|
| Flat (납작) | (D·0.6/2, 2dp) | = h | 0.0 | 0 |
| Sphere (구) | (D/2, D/2) | = D/2 | 1.0 | 최대 |
| Settled card (L2) | (W/2, H/2) | settledCornerDp | 0.25 (bevel만) | 0.15 |
| Full screen (L3) | (화면/2) | 0 → 엣지 글로우 | 0.1 | 0 |

- **elevation**은 셰이더에서 노멀을 재구성할 때 z 성분의 스케일이다. 1이면 완전한 반구 셰이딩, 0이면 평면 그라데이션이 된다. 그래서 "물성은 유지한 채 퍼져서 안착"하는 느낌을 연속적으로 낼 수 있다.
- 노멀 재구성: SDF 내부 거리 `d`와 경계까지의 정규화 거리로 높이장 `h = elevation · sqrt(1 − (1 − d/bevel)²)`을 만들고, 그라디언트로 노멀을 얻는다(bevel = 구일 때는 반지름, 카드일 때는 `settledThicknessDp`).

### 5-2. 한 사이클 & 레벨 타임라인 (기본값, speed = 1)

```
t(ms)   0        800             3400      4200   4550            7000      7800          10400
        │ L1 enter│ L1 hold (부유) │ L2 enter │settle│ L2 hold        │ L3 enter│ L3 hold      │→ L1
        │flat→구 │ bob ±3dp       │구 이동·확산→카드│ list ×3 stagger │ 카드→전체 │ 엣지 글로우   │
```

| 구간 | 내용 | 이징 |
|---|---|---|
| L1 enter (800) | 상단 중앙 flat pill → 위로 떠오르며 구(elevation 0→1), 그림자 0→최대 | in(emphasized-decel) |
| L1 hold (2600) | 부유 bob(sin, 2400ms 주기) + 하이라이트 드리프트 | linear/sin |
| L2 enter (800) | 구가 목표 위치(화면 중하단)로 이동하며 가로로 퍼짐 → 카드로 morph, elevation 1→0.25, 그림자 축소. 영상의 "out 300 → in 400" 리듬을 **확산 300(가속) → 안착 500(감속)**으로 재사용 | out → in |
| L2 settle (350) | 미세 오버슛 없이 감속 정착. 이 시점에 리스트 아이템 생성 시작 | settle |
| 컴포넌트 생성 | 아이템 3개, 각 240ms 페이드+8dp 슬라이드업, 60ms stagger | settle |
| L2 hold (≈2450) | 카드 내부 하이라이트 드리프트 | — |
| L3 enter (800) | 리스트 아이템이 먼저 페이드아웃(시차 −150ms) → 카드가 전체 화면으로 확장, corner→0, 중앙은 투명도를 낮춰 콘텐츠를 가리지 않는 엣지 글로우만 남김 | out → in |
| L3 hold (2600) | 엣지 글로우 숨쉬기(opacity ±10%) | sin |
| L3→L1 (800, 다음 루프의 L1 enter와 겹침) | 전체 화면 → 상단 중앙으로 수축하며 구로 복귀 | out → in |

- `transitionStyle = "phase"`이면 레벨 전환 중간에 영상과 같은 **위상 회전 전환**(색면이 회전하며 빠졌다 들어옴)을 넣는다. 셰이더의 `uPhase` 각도 uniform으로 구현한다.
- 모든 ms 값은 `CueSpec.motion.levels`와 `speed`로 조절할 수 있다.

### 5-3. 컬러

- 스톱 1–3개, 스톱마다 HEX/이름/위치를 편집한다. 칩 드래그로 위치를 바꾸고, 클릭하면 컬러 피커가 열린다.
- 셰이더에는 `uStops[3]`(vec4: rgb + position)와 `uStopCount`를 넘긴다. 보간은 **OKLab 공간**에서 한다. 영상처럼 탁해지지 않는 보라→하늘 그라데이션을 위해서이며, sRGB 보간 토글도 제공한다.
- 기본 프리셋 3종(§1-2)과 "현재 설정을 프리셋으로 저장" 기능을 둔다(localStorage).

### 5-4. 물성(Material) 타입 — 임시 정의 (A1·V9)

| 타입 | 시각적 특징 | 셰이더 구성 요소 | 타입별 파라미터 | AGSL 가능 여부 |
|---|---|---|---|---|
| **Soft Glow** (기본, 영상 기준) | 매트한 그라데이션 볼륨, 밝은 하단 림, 외곽 번짐 | 노멀 기반 그라데이션 매핑 + rim light + 외곽 가우시안 근사 글로우(SDF falloff) | glowRadius, rimStrength, softness | ◎ |
| **Frosted Glass** | 반투명, 뒤 배경이 흐리게 굴절되어 보임, 프레넬 테두리 | 배경 샘플링(`uniform shader content`) + 노멀 오프셋 굴절 + 다중 샘플 블러 근사 + fresnel | refraction, frost, tintOpacity, fresnel | ◎ (RenderEffect 경로, API 33+) / 웹은 앱 목업을 텍스처로 렌더해 샘플링 |
| **Liquid Metal** | 크롬처럼 그라데이션이 반사 띠로 나타남, 강한 스펙큘러 | 반사 벡터 → 그라데이션 LUT(matcap 방식, 텍스처 없이 수식) + 블린-퐁 스펙큘러 | reflectivity, specularSize, bandCount | ◎ |
| **Pearl** | 무지갯빛(iridescent)이 시야각에 따라 변함, 부드러운 광택 | 박막 간섭 근사(`cos(n·v · k)`로 색상 회전) + 스톱 그라데이션 블렌드 | iridescence, sheen, hueShift | ◎ |
| **Matte Clay** | 무광, 미세 노이즈 질감, 부드러운 앰비언트 오클루전 | 람베르트 + 값 노이즈(해시 기반, 텍스처 불필요) + 가장자리 AO | grain, roughness, aoStrength | ◎ |

- 모든 타입은 **텍스처를 쓰지 않고 수식만으로** 구현한다. AGSL로 옮길 때 리소스 의존성을 없애기 위해서다.
- Reference effect 이미지가 제공되면 Phase 0의 "레퍼런스 대조"에서 타입명과 파라미터를 교체·추가한다. 셰이더는 타입별 `material_*.glsl` 함수로 분리되어 있어 교체 비용이 낮다.

### 5-5. 크기

- 구 지름(dp), flat 폭 비율/두께, 안착 카드 높이/두께/모서리, 여백.
- 모든 값은 dp로 저장한다. 웹 프리뷰는 `캔버스 실제 px / 360`을 배율로 써서 dp를 px로 변환한다.

---

## 6. Phase별 계획

> 각 Phase는 **목표 → 세부 작업 → 산출물 → 완료 기준**으로 구성하고, 참조 자료와 예상 기간을 함께 적는다.

---

### Phase 0 — 요구사항 정리 · 가정 확정 · 기술 스파이크 (예상 4–5일)

**목표**
불확실성이 가장 큰 두 가지를 구현 전에 검증한다. ①웹 GLSL과 Android AGSL에서 같은 셰이더가 같은 그림을 내는지, ②Reference video의 리듬을 데이터로 정확히 옮길 수 있는지. 그리고 §0의 가정을 확정한다.

**세부 작업**
1. §0 가정 목록 리뷰 미팅 → 각 항목 확정/수정, 결정 로그(`docs/decisions.md`) 작성
2. Reference video 정밀 분석 문서화: §1 수치 재검증(30fps), 이징 곡선 피팅(전환 구간 채색 비율 곡선 → 베지어 4숫자)
3. **셰이더 동형성 스파이크**
   - Soft Glow 구 하나를 GLSL ES 3.0(WebGL2)과 AGSL로 각각 작성
   - GLSL ↔ AGSL 차이 목록 작성(`vec`↔`float2/half4`, `main(float2 fragCoord)` 시그니처, `texture` → `shader.eval`, 정밀도 한정자, 루프/배열 제약, 좌표계 y축 방향)
   - 차이를 흡수할 **템플릿 방식 결정**: (a) 매크로 기반 공통 소스(`#define vec2 float2` 류), (b) 미니 트랜스파일러(정규식 치환 + 검증), (c) 셰이더를 TS DSL로 작성해 두 언어로 출력 → **(a) 우선, 한계가 있으면 (b)**로 결정
   - Android 하네스 앱(`android-harness/`)에서 실제 기기/에뮬레이터(API 33, 34)로 렌더링 후 웹 스크린샷과 나란히 비교
4. 타임라인 엔진 PoC: 키프레임 테이블 → `sampleFrame(t)` → Compose `keyframes` 출력 가능성 확인
5. 기술 스택 확정(§3) 및 리포 스캐폴딩(Vite + React + TS, ESLint/Prettier, Vitest, Playwright, GitHub Actions에 Android 빌드 잡 포함)
6. `CueSpec` v1 스키마 초안(Zod) + 기본값 + 프리셋 3종
7. (이미지 자료 제공 시) **레퍼런스 대조**: Reference effect → 물성 타입 확정, Reference level → 레벨 레이아웃 확정, App examples → 영상② 목업 방향 확정

**산출물**
- `docs/decisions.md` (가정 확정 로그), `docs/reference-analysis.md` (영상 수치·이징)
- 셰이더 스파이크 결과: 웹/Android 비교 스크린샷 + GLSL↔AGSL 변환 규칙표
- `src/spec/cueSpec.ts` (Zod 스키마, 기본값, 프리셋)
- 리포 스캐폴드 + CI 파이프라인(웹 빌드, 단위 테스트, Android 하네스 `assembleDebug`)

**완료 기준**
- [ ] §0의 모든 가정이 확정 또는 수정 상태로 기록됨
- [ ] 같은 Soft Glow 구가 웹과 Android(API 33)에서 **육안상 동일**(색차 ΔE2000 평균 < 3, 중심·림 3점 샘플 기준)
- [ ] 이징 4숫자로 재생한 전환 곡선이 레퍼런스 측정값과 **±50ms 이내**로 일치
- [ ] CI에서 웹 빌드와 Android 하네스 빌드가 모두 통과

**참조 자료**: Reference video(타이밍·셰이딩), (제공 시) Reference effect·level·App examples 1차 대조

---

### Phase 1 — 레이아웃 뼈대 (예상 4일)

**목표**
툴의 전체 프레임을 완성한다. 탭바, 좌우 분할, 다크/라이트 모드, 프리뷰 3뷰 전환, 공통 재생 시계까지. 이 단계의 프리뷰 캔버스에는 Phase 0 스파이크의 정지 구만 들어간다.

**세부 작업**
1. **디자인 토큰**: 색(라이트/다크), 타이포(라벨은 영상처럼 작은 회색 산세리프. 예: Inter/IBM Plex Sans), 간격, 반경, 그림자를 CSS 변수로 정의
2. **상단 탭바**
   - 좌: 타이틀 "AI Visual Cue Studio"
   - 중앙: `AI Visual Cue` / `About` 탭(키보드 접근성, `role="tablist"`, 활성 인디케이터 슬라이드)
   - 우: ☀︎/☾ 토글(아이콘 회전 전환, `aria-pressed`, `prefers-color-scheme` 초기화 + localStorage)
3. **About 탭**: 정적 콘텐츠 페이지(소개, 사용법, 생성 코드 요구사항, 버전)
4. **본문 분할**: 좌 1/3 컨트롤 패널(min 360 / max 480), 우 2/3 프리뷰. 1024px 미만에서는 하단 시트로 전환
5. **컨트롤 패널 서브탭** `Design` / `Code` 빈 껍데기와 스크롤 영역
6. **프리뷰 영역**
   - 360×780dp 비율 고정 캔버스 컴포넌트 `<DeviceFrame>`(얇은 폰 베젤 옵션, 안전영역 가이드 토글)
   - 3뷰 캐러셀: ① 영상①+② 나란히 ② 영상① 단독 ③ 영상② 단독
   - 전환: ←/→ 버튼, 키보드, 포인터 드래그(관성·스냅), 페이지 인디케이터, 뷰 이름 라벨
   - **공유 재생 시계**(`useCueClock`): rAF 기반 단일 시계, 재생/일시정지, 속도, 스크러버, 탭 비활성 시 일시정지(`visibilitychange`)
7. 성능 기준선: 2개 WebGL 컨텍스트 대신 **단일 WebGL 컨텍스트 + 시저/뷰포트 분할** 또는 OffscreenCanvas 공유 여부 결정(뷰 1에서 캔버스 2개)

**산출물**
- 동작하는 앱 셸(탭, 테마, 분할, 3뷰 캐러셀, 재생 컨트롤)
- `src/ui/tokens.css`, `src/ui/layout/*`, `src/preview/Carousel.tsx`, `src/preview/useCueClock.ts`

**완료 기준**
- [ ] 1280×800, 1440×900, 1920×1080에서 레이아웃이 깨지지 않고 캔버스 비율 360:780이 유지됨
- [ ] 다크/라이트 전환 시 깜빡임(FOUC) 없음, 새로고침 후에도 유지
- [ ] 3뷰 전환이 버튼/키보드/드래그 모두로 동작하고 루프 재생이 끊기지 않음(뷰 전환 중에도 시계 연속)
- [ ] 뷰 1의 두 캔버스가 같은 `t`를 사용함(디버그 오버레이로 확인)
- [ ] Lighthouse 접근성 90+

**참조 자료**: Reference video(라벨 타이포·여백 톤 참고), (제공 시) App examples(디바이스 프레임 비율 확인)

---

### Phase 2 — 비주얼 큐 코어 모션 (예상 8–9일, 가장 큰 Phase)

**목표**
`CueSpec` 기본값만으로 **납작 → 구 → 부유 → 안착 → 컴포넌트 생성** 사이클과 **L1→L2→L3→L1 무한 루프**를 영상① 뷰에서 완성한다. 물성은 Soft Glow 1종이다.

**세부 작업**
1. **셰이더 템플릿 v1** (`shaders/cue.template.glsl`)
   - 공통부: 둥근 사각형 SDF, 높이장 → 노멀, OKLab 그라데이션(1–3 스톱), 하이라이트 코어 오프셋, 안티앨리어싱(fwidth 대체로 dp 기반 고정 폭 → AGSL 호환)
   - Soft Glow 물성 함수
   - 그림자: SDF 기반 소프트 섀도(별도 패스 없이 같은 셰이더에서 `offsetY`만큼 이동한 SDF로 계산)
   - `uPhase` 위상 전환(옵션)
   - 웹용 빌드: 템플릿 → GLSL ES 3.0 문자열(빌드 타임 생성 + 런타임 uniform만 변경)
2. **타임라인 엔진** (`src/motion/`)
   - `deriveTimeline(spec)`: 레벨 타이밍 → 키프레임 배열(시간, 상태, 이징)
   - `sampleFrame(spec, tl, t)`: 형태 파라미터 보간(center, halfSize, corner, elevation, shadow), 부유 bob, 하이라이트 드리프트, 컴포넌트 진행도
   - 이징 함수: cubic-bezier 솔버(Compose `CubicBezierEasing`과 같은 알고리즘, 오차 < 1e-3)
3. **렌더러** (`src/preview/CueRenderer.ts`): WebGL2 셋업, uniform 바인딩, DPR 대응, 컨텍스트 손실 복구
4. **컴포넌트 레이어(DOM)**: 영상① 단독 뷰에서 안착 카드 내부에 skeleton 리스트 아이템 3개를 stagger 등장/퇴장. 카드 bounds는 `sampleFrame` 결과로 위치를 동기화
5. **레벨별 연출 디테일**
   - L1: 상단 중앙(상태바 아래 24dp) flat pill에서 떠오름
   - L2: 구 이동 경로는 직선이 아니라 약한 아크(위로 4dp 볼록), 가로 확산이 세로 확산보다 100ms 먼저(질량감)
   - L3: 전체 화면 확장 후 중앙 투명화 → 엣지 글로우
   - L3→L1: 상단으로 수축
6. **디버그 오버레이**: 현재 레벨/구간/t, 키프레임 마커 표시(개발 전용 토글)
7. 레퍼런스 대조 세션: 영상과 프리뷰를 나란히 놓고 리듬(유지 대비 전환 비율, 가속/감속 느낌) 확인

**산출물**
- `shaders/cue.template.glsl`, `shaders/materials/softGlow.glsl`
- `src/motion/{timeline,sample,easing}.ts` + 단위 테스트
- 영상① 뷰에서 무한 루프 재생되는 코어 모션
- `docs/motion-spec.md` (최종 키프레임 표)

**완료 기준**
- [ ] 기본값에서 루프 길이 = Σ(enter+hold) ≈ 10.4s이고, 루프 경계(L3→L1)에서 **튐(pop) 없음**(t=0과 t=loop 프레임 픽셀 diff < 1%)
- [ ] 형태 morph 중 셰이딩이 형태를 따라감(구일 때 반구 셰이딩, 카드일 때 bevel 셰이딩), 경계에 계단(aliasing)이나 깜빡임 없음
- [ ] 리스트 아이템이 카드 안착 완료 후에만 등장하고 L3 진입 전에 사라짐
- [ ] `sampleFrame` 단위 테스트: 키프레임 경계값, speed 0.25/2 배속, 레벨 timing 변경 케이스 통과
- [ ] M1 MacBook Air / 중급 Windows 노트북(내장 GPU) 크롬에서 뷰 1(캔버스 2개) **60fps 유지**(프레임 드랍 < 2%)
- [ ] 디자이너 리뷰: "영상의 리듬과 같은 느낌" 승인

**참조 자료**: **Reference video(타이밍·이징·셰이딩의 주 기준)**, (제공 시) Reference level(레벨별 위치·크기), Reference effect(Soft Glow 톤)

---

### Phase 3 — 컨트롤 패널 연결 (예상 6–7일)

**목표**
Design 탭의 모든 컨트롤이 `CueSpec`을 수정하고, 프리뷰가 **실시간(같은 프레임)**으로 반영되게 한다. 물성 5종을 모두 구현한다.

**세부 작업**
1. **컬러 섹션**
   - 그라데이션 에디터: 영상식 세로선 + 칩 UI. 칩 드래그 = position, 클릭 = 컬러 피커(react-colorful + HEX 입력 + 스포이드는 `EyeDropper` API 지원 시)
   - 스톱 추가(+, 최대 3)/삭제(−, 최소 1), 이름 편집
   - 프리셋 3종 칩(Azure / Blossom / Indigo-Bloom) + 사용자 프리셋 저장
   - gradientType, angle, 하이라이트 위치(2D 패드), OKLab/sRGB 토글
2. **물성 섹션**
   - 타입 선택: 썸네일 카드 5개(각 타입을 정지 구로 렌더한 실시간 미니 프리뷰)
   - 타입별 파라미터 슬라이더(§5-4), 타입 전환 시 파라미터 기본값 로드
   - 셰이더 구현: Frosted Glass(웹은 영상② 목업 DOM을 `html-to-image` 대신 **목업을 셰이더 내부 절차적 패턴 또는 사전 렌더 텍스처**로 대체할지 Phase 0 결정에 따름), Liquid Metal, Pearl, Matte Clay
3. **크기 섹션**: 구 지름, flat 폭/두께, 안착 높이/두께/모서리, 여백. 숫자 입력 + 슬라이더 + 단위(dp) 표시
4. **모션 섹션(가능하면 항목)**: 전체 속도, 그림자 강도/블러/오프셋, 부유 진폭/주기, 하이라이트 드리프트, 전환 스타일(morph/phase), 레벨별 enter/hold(ms), stagger, 이징 프리셋(Reference / Standard / Snappy) + 고급 베지어 편집
5. **공통 UX**: 섹션 접기/펼치기, 값 리셋(섹션별/전체), undo/redo(⌘Z/⇧⌘Z), 변경된 값 표시 점(●)
6. **상태 영속성**: localStorage 자동 저장, URL 공유 링크, JSON 내보내기/불러오기(Zod 검증)
7. 성능: 슬라이더 드래그 중 리렌더 최소화(셀렉터 구독, uniform만 갱신하고 셰이더 재컴파일 없음. 물성 타입 변경 시에만 프로그램 교체. 5종을 미리 컴파일해 캐시)

**산출물**
- `src/panel/design/{ColorSection,MaterialSection,SizeSection,MotionSection}.tsx`
- `shaders/materials/{frostedGlass,liquidMetal,pearl,matteClay}.glsl`
- `src/spec/persistence.ts`(localStorage, URL, 파일)

**완료 기준**
- [ ] 모든 컨트롤 조작이 **16ms 이내** 프리뷰에 반영됨(드래그 중 끊김 없음)
- [ ] 스톱 1/2/3개, 물성 5종 × 기본 프리셋 3종 = 45개 조합 스크린샷이 시각 회귀 기준으로 등록됨
- [ ] JSON 내보내기 → 불러오기 왕복 시 `CueSpec` 완전 동일(deep equal)
- [ ] 잘못된 JSON(범위 밖, 스톱 4개 등)이 필드 단위 에러로 표시되고 앱이 깨지지 않음
- [ ] undo/redo 50단계 동작
- [ ] 물성 타입 전환 시 셰이더 재컴파일로 인한 프레임 멈춤 없음(사전 컴파일)

**참조 자료**: Reference video(프리셋 팔레트·그라데이션 에디터 UI), **(제공 시) Reference effect(물성 5종 룩 대조 및 교체)**

---

### Phase 4 — Compose 코드 생성기 + Code 탭 (예상 7–8일)

**목표**
현재 `CueSpec`으로 **복사·붙여넣기만 하면 Android에서 동작하는** 단일 Kotlin 파일을 생성하고, 생성 코드가 실제로 컴파일되고 웹 프리뷰와 같은 결과를 낸다는 것을 자동으로 검증한다.

**세부 작업**
1. **셰이더 변환 파이프라인**: Phase 0에서 정한 방식으로 `cue.template.glsl` + 선택된 물성 → AGSL 문자열. 사용하지 않는 물성 코드는 제거(데드코드 제거)하고, uniform 기본값을 상수로 인라인할지 uniform으로 둘지 옵션 제공
2. **Kotlin 코드 생성기** (`src/codegen/compose/`)
   - 템플릿 엔진: 태그드 템플릿 리터럴 + 들여쓰기 유틸(외부 템플릿 엔진 없음)
   - 생성 파일 구성(§10-2 골격 참고):
     - `AiVisualCueSpec` data class(현재 값이 기본값으로 박힘) + 프리셋 상수
     - `CueLevel` enum(L1/L2/L3) + **두 가지 사용 모드**: ① `AiVisualCueDemoLoop()`(툴과 동일한 무한 루프) ② `AiVisualCue(level: CueLevel)`(앱에서 상태로 제어. 레벨 변경 시 해당 전환 재생)
     - 타임라인: `Animatable` + `CubicBezierEasing(…)`(웹과 같은 4숫자), 무한 루프는 `rememberInfiniteTransition` 또는 `withFrameNanos` 기반 시계
     - 렌더링: `Modifier.drawWithCache { val shader = RuntimeShader(AGSL); val brush = ShaderBrush(shader); onDrawBehind { shader.setFloatUniform(...); drawRect(brush) } }`
     - Frosted Glass 선택 시: `graphicsLayer { renderEffect = RenderEffect.createRuntimeShaderEffect(shader, "content").asComposeRenderEffect() }` 경로
     - 컴포넌트 생성 슬롯: `content: @Composable (progress: Float) -> Unit` 슬롯 + 기본 skeleton 리스트 구현
     - **API 분기**: `if (Build.VERSION.SDK_INT >= 33) ShaderCue(...) else FallbackCue(...)`. 폴백은 `Brush.radialGradient` + `Modifier.shadow` + 모서리 보간 + (31+) `Modifier.blur`
     - dp 단위 그대로 사용, `LocalDensity`로 px 변환해 uniform 전달
     - `@Preview` 함수 포함
   - 상단 주석: 생성 일시, 툴 버전, 복원용 `CueSpec` JSON(한 줄 압축), 요구사항(minSdk 26, AGSL은 33+, Compose BOM 버전)
3. **Code 탭 UI**
   - Shiki 하이라이트(라이트/다크 테마 연동), 줄 번호, 접기
   - **복사 버튼**(클립보드, 성공 토스트), `.kt` 파일 다운로드, JSON 다운로드
   - 서브탭 또는 토글: `Kotlin (전체)` / `AGSL만` / `Spec JSON`
   - 값 변경 시 diff 하이라이트(변경된 줄 1초 강조), 생성은 디바운스 100ms
   - 경고 영역: 예) "Frosted Glass는 API 33 미만에서 Soft Glow로 폴백됩니다"
4. **생성 코드 검증 자동화** (`android-harness/`)
   - CI 잡: 대표 `CueSpec` 픽스처 N개(물성 5종 × 스톱 1/3개 + 극단값) → 코드 생성 → 하네스 프로젝트에 주입 → `./gradlew assembleDebug` + ktlint
   - AGSL 런타임 컴파일 검증: 에뮬레이터(API 34, GPU swiftshader) 계측 테스트로 `RuntimeShader(src)` 생성 시 예외 없음 확인
   - 시각 동등성: 에뮬레이터에서 고정 `t`(예: L1 hold 중간, L2 hold 중간, L3 hold 중간) 스크린샷 ↔ Playwright 웹 스크린샷 비교(ΔE 평균 < 3, 형태 IoU > 0.97)
   - 참고: Paparazzi/layoutlib는 RuntimeShader 렌더링 지원이 불확실하므로 **에뮬레이터 스크린샷을 기준**으로 한다(Phase 0 스파이크에서 재확인)
5. 생성기 스냅샷 테스트(Vitest): 픽스처별 생성 결과 고정, 의도치 않은 출력 변경 감지

**산출물**
- `src/codegen/{agsl,compose,index}.ts` + 스냅샷 테스트
- Code 탭 UI
- `android-harness/` + CI 워크플로(`.github/workflows/android-verify.yml`)
- `docs/generated-code-guide.md` (앱 개발자용 사용 가이드: 붙여넣기, 레벨 제어, 슬롯 커스터마이즈)

**완료 기준**
- [ ] 모든 픽스처의 생성 코드가 **수정 없이** 컴파일되고 ktlint를 통과
- [ ] API 33/34 에뮬레이터에서 AGSL 컴파일 에러 0건, API 26/30 에뮬레이터에서 폴백 경로 정상 표시
- [ ] 웹 ↔ Android 시각 동등성 기준(ΔE < 3, IoU > 0.97) 통과
- [ ] 같은 `CueSpec` → 같은 코드(결정적 출력), 생성 시간 < 30ms
- [ ] 생성 코드를 새 Compose 프로젝트(Empty Activity 템플릿)에 붙여넣고 `AiVisualCueDemoLoop()`만 호출했을 때 툴과 같은 루프가 재생됨(수동 확인, Pixel 실기기 1대 이상)
- [ ] 실기기 성능: Pixel 6급 기기에서 60fps 유지, 셰이더 1프레임 GPU 시간 < 4ms(GPU 프로파일러)

**참조 자료**: Reference video(이징·타이밍 수치를 코드 상수로 반영), (제공 시) App examples(슬롯 기본 리스트 형태)

---

### Phase 5 — 영상② 앱 적용 프리뷰 · 폴리시 · QA (예상 6일)

**목표**
비주얼 큐가 **실제 앱 UI에 녹아드는 모습**(영상②)을 완성하고, 툴 전체의 완성도와 품질을 출시 수준으로 끌어올린다.

**세부 작업**
1. **영상② 앱 목업** (DOM/CSS, 텍스트 없는 와이어 수준. 단 앱다운 밀도)
   - 가상 앱 화면: 상태바, 앱바, 콘텐츠 피드/카드, 하단 입력창(AI 에이전트 호출 지점)
   - L1: 상단(앱바 아래)에 구 생성, 뒤 콘텐츠는 살짝 디밍(10%)
   - L2: 구가 콘텐츠 영역으로 퍼져 **응답 리스트 카드**로 안착 → 리스트 셀 3개(아이콘 + 2줄 skeleton) stagger 생성, 기존 콘텐츠는 아래로 밀림(레이아웃 시프트 애니메이션)
   - L3: 전체 화면 확산 → 엣지 글로우 + 풀스크린 에이전트 모드(배경 블러)
   - `preview.appTheme`(light/dark/auto)에 따라 목업 테마 전환
   - Frosted Glass 물성이 목업 콘텐츠를 실제로 굴절시키도록 목업을 텍스처로 공급(Phase 3 결정 방식 적용)
2. **뷰 1 동기화 폴리시**: 영상①·② 같은 시계, 라벨(“① Visual Cue”, “② In-App”), 뷰 간 전환 애니메이션
3. **UX 폴리시**
   - 빈 상태/에러 상태(WebGL 미지원 안내, 컨텍스트 손실 복구 메시지)
   - 툴팁(각 파라미터 설명), 키보드 단축키 시트(`?`)
   - 프리뷰 캡처: 현재 프레임 PNG 저장, (선택) 루프 WebM 녹화(`MediaRecorder` + `canvas.captureStream`)
   - About 탭 콘텐츠 완성
4. **QA**
   - 크로스브라우저: Chrome, Edge, Safari 17+, Firefox 최신(WebGL2 차이 확인)
   - 시각 회귀: Phase 3 45조합 + 영상② 3레벨 × 테마 2종
   - 접근성: 키보드만으로 전체 조작, 스크린리더 라벨, `prefers-reduced-motion` 시 프리뷰 기본 일시정지 + 안내
   - 성능 회귀: 뷰 1 60fps, 메모리 누수(10분 연속 재생 후 힙 증가 < 10%)
   - 생성 코드 회귀: Phase 4 CI 전체 통과
5. 문서화 및 배포: README, 사용 가이드, 정적 호스팅 배포, 버전 태깅 v1.0

**산출물**
- 영상② 앱 목업 프리뷰(`src/preview/app-mock/*`)
- QA 리포트(`docs/qa-report.md`: 브라우저 매트릭스, 성능 수치, 알려진 이슈)
- 배포된 v1.0 웹앱 URL, 사용자/개발자 가이드

**완료 기준**
- [ ] 영상②에서 L1→L2→L3→L1 루프 중 **목업 UI와 큐의 레이어 순서, 위치, 시차가 어긋나지 않음**(안착 카드와 리스트 셀 정렬 오차 < 1dp)
- [ ] 디자이너 최종 리뷰 승인(체크리스트: 물성 유지감, 확산의 자연스러움, 안착감, 루프 이음새)
- [ ] 지원 브라우저 4종에서 치명적/주요 버그 0건
- [ ] Lighthouse 성능 85+, 접근성 95+
- [ ] Phase 4 생성 코드 검증 CI 그린

**참조 자료**: **(제공 시) App examples(목업 구조의 주 기준)**, Reference level(앱 내 레벨별 점유 영역), Reference video(최종 리듬 대조)

---

### Phase 6 (선택) — 확장 (일정 미정)

- 레벨 수 가변(L1–Ln), 사용자 정의 키프레임 편집기(타임라인 UI)
- 생성 코드 → `CueSpec` 역파싱(헤더 JSON 기반 복원은 v1에 포함, 코드 자체 파싱은 확장)
- iOS(SwiftUI + Metal) 코드 출력 — 셰이더 템플릿 구조를 재사용
- Lottie/MP4 내보내기(마케팅 용도)
- 팀 프리셋 라이브러리(백엔드 필요)

---

### 일정 요약

| Phase | 내용 | 예상 기간 | 누적 |
|---|---|---|---|
| 0 | 요구사항·가정 확정, 셰이더 동형성 스파이크 | 4–5일 | 1주 |
| 1 | 레이아웃 뼈대 | 4일 | ~2주 |
| 2 | 코어 모션 | 8–9일 | ~3.5주 |
| 3 | 컨트롤 패널 + 물성 5종 | 6–7일 | ~5주 |
| 4 | Compose 코드 생성기 + 검증 | 7–8일 | ~6.5주 |
| 5 | 영상② · 폴리시 · QA | 6일 | **~7.5주** |

> Phase 3과 Phase 4는 일부 병렬 진행이 가능하다(생성기 골격은 Phase 2 직후 착수 가능). 2인이면 약 5.5주로 단축할 수 있다.

---

## 7. 참조 자료 × Phase 매핑

| 자료 | 상태 | P0 | P1 | P2 | P3 | P4 | P5 |
|---|---|---|---|---|---|---|---|
| Reference video | ✅ 분석 완료(§1) | 수치 재검증·이징 피팅 | 라벨/톤 | **주 기준**: 루프·이징·셰이딩 | 프리셋 팔레트, 그라데이션 에디터 UI | 이징·타이밍 상수 | 최종 리듬 대조 |
| Reference effect | ⏸ 이번 계획에서 제외(A1) | 제공 시 물성 타입 확정 | — | Soft Glow 톤 | **주 기준**: 물성 5종 룩 | — | — |
| Reference level | ⏸ 이번 계획에서 제외(A1) | 제공 시 레벨 레이아웃 확정 | — | 레벨별 위치·크기 | — | — | 앱 내 레벨 점유 영역 |
| App examples | ⏸ 이번 계획에서 제외(A1) | 제공 시 목업 방향 확정 | 디바이스 프레임 | — | — | 슬롯 기본 리스트 형태 | **주 기준**: 영상② 목업 |

---

## 8. 리스크와 대응

| 리스크 | 영향 | 가능성 | 대응 |
|---|---|---|---|
| GLSL ↔ AGSL 표현 차이(정밀도, `fwidth` 미지원, 루프 제약, 좌표 y축)로 웹과 기기 결과가 다름 | 높음 | 중 | Phase 0 스파이크에서 최우선 검증. 템플릿은 AGSL 가능 기능의 **교집합만** 사용. CI 시각 동등성 테스트 |
| API 33 미만 기기 비중이 높아 폴백 품질이 문제됨 | 중 | 중 | 폴백도 같은 형태·모션·그라데이션 보장. 물성만 근사. 툴 Code 탭에 폴백 미리보기 토글 추가(Phase 4) |
| Frosted Glass의 배경 샘플링(웹: DOM → 텍스처)이 무겁거나 부정확함 | 중 | 높음 | 웹은 목업을 Canvas2D로 직접 그려 텍스처로 공급(DOM 캡처 회피). 안 되면 Glass를 "목업 정적 스냅샷 굴절"로 제한 |
| 2개 캔버스 동시 60fps 미달(저사양 GPU) | 중 | 중 | 단일 컨텍스트 뷰포트 분할, DPR 상한 2, 품질 토글(Low = 샘플 수 감소) |
| 레퍼런스 이미지(effect/level/app) 미반영으로 이후 방향 변경 | 중 | 중 | 물성·레벨·목업을 데이터/모듈로 분리해 교체 비용 최소화. Phase 0 말에 대조 체크포인트 |
| 생성 코드가 사용자 프로젝트의 Compose 버전과 충돌 | 낮음 | 중 | 안정 API만 사용(`ShaderBrush`, `CubicBezierEasing`, `drawWithCache`), 헤더에 테스트한 BOM 버전 명시, CI에서 BOM 2개 버전 매트릭스 |
| "영상 리듬과 같은 느낌"이 주관적이라 합의 지연 | 중 | 중 | 영상과 프리뷰를 나란히 놓는 비교 모드(디버그) 제공, 수치 기준(§1-4) 병행 |

---

## 9. 범위 밖 / 추후 과제

- Code 탭 직접 편집 → 프리뷰 역반영(round-trip)
- 다중 사용자 협업, 클라우드 저장, 로그인
- 3D 메시 기반 렌더링(three.js 등). AGSL 동형성을 위해 의도적으로 제외
- iOS/Flutter/웹 컴포넌트 코드 출력(Phase 6 후보)
- 사운드/햅틱 큐

---

## 10. 부록

### 10-1. 예상 폴더 구조

```
ai-visual-cue-studio/
├─ src/
│  ├─ app/            # App shell, 라우팅(탭), 테마
│  ├─ ui/             # 디자인 토큰, 공통 컴포넌트
│  ├─ spec/           # CueSpec 스키마(Zod), 기본값, 프리셋, 영속성
│  ├─ store/          # Zustand 스토어 + undo/redo
│  ├─ motion/         # timeline, sampleFrame, easing (순수 함수)
│  ├─ preview/        # Carousel, DeviceFrame, CueRenderer(WebGL2), useCueClock
│  │  └─ app-mock/    # 영상② 앱 목업
│  ├─ panel/
│  │  ├─ design/      # Color/Material/Size/Motion 섹션
│  │  └─ code/        # Code 탭(하이라이트, 복사, 다운로드)
│  └─ codegen/        # agsl.ts, compose.ts, templates/
├─ shaders/
│  ├─ cue.template.glsl
│  └─ materials/      # softGlow / frostedGlass / liquidMetal / pearl / matteClay
├─ android-harness/   # 생성 코드 컴파일·렌더 검증용 Gradle 프로젝트
├─ tests/             # vitest(단위·스냅샷), playwright(E2E·시각 회귀)
└─ docs/              # decisions, reference-analysis, motion-spec, qa-report, guides
```

### 10-2. 생성 Kotlin 파일 골격 (형태 참고용, 구현 아님)

```kotlin
// Generated by AI Visual Cue Studio v1.0 — 2026-xx-xx
// minSdk 26 · AGSL path requires API 33+ · Tested with Compose BOM 20xx.xx
// spec: {"version":1,"colors":{...}}   ← 툴에 다시 불러올 수 있는 JSON

package com.example.aicue

data class AiVisualCueSpec(/* 현재 값이 기본값으로 들어감 */)

enum class CueLevel { L1, L2, L3 }

private const val CUE_AGSL = """ /* 템플릿에서 변환된 AGSL */ """

private val EaseOut    = CubicBezierEasing(0.55f, 0f, 1f, 0.45f)
private val EaseIn     = CubicBezierEasing(0.05f, 0.7f, 0.1f, 1f)
private val EaseSettle = CubicBezierEasing(0.2f, 0f, 0f, 1f)

@Composable fun AiVisualCueDemoLoop(modifier: Modifier = Modifier, spec: AiVisualCueSpec = AiVisualCueSpec())
@Composable fun AiVisualCue(level: CueLevel, modifier: Modifier = Modifier, spec: AiVisualCueSpec = AiVisualCueSpec(),
                            content: @Composable (progress: Float) -> Unit = { DefaultSkeletonList(it) })

// 내부: API 33+ → ShaderCue (RuntimeShader + ShaderBrush / RenderEffect)
//       API 26–32 → FallbackCue (Brush.radialGradient + shadow + 모서리 보간)
@Preview @Composable private fun AiVisualCuePreview()
```

### 10-3. 용어

| 용어 | 의미 |
|---|---|
| 영상① | 비주얼 큐 단독 프리뷰(중립 배경) |
| 영상② | 앱 UI 목업 위에 적용된 프리뷰 |
| SDF | Signed Distance Field. 형태를 거리 함수로 표현해 연속 morph에 사용 |
| elevation | 셰이더의 볼륨감 계수(0 = 평면, 1 = 완전한 구) |
| 동형성(parity) | 웹 프리뷰와 Android 결과가 같은 데이터·같은 셰이더에서 나와 시각적으로 일치하는 성질 |

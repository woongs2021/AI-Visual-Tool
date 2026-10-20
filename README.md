# AI Visual Cue Studio

AI 에이전트가 활성화될 때 화면에 나타나는 **비주얼 큐(AI Visual Cue)**의 컬러·물성·크기·모션을 웹에서 조절하고, 그 결과를 **Kotlin + Jetpack Compose 코드**로 내보내는 디자인 툴입니다.

### ▶ 바로 열기: **https://woongs2021.github.io/AI-Visual-Tool/**

설치나 빌드 없이 브라우저에서 바로 동작합니다. WebGL2를 지원하는 최신 Chrome, Edge, Safari, Firefox에서 열 수 있습니다.

---

## 주요 기능

- **프리뷰 3개 뷰**: 전체 비교 / ① 비주얼 큐 단독 / ② 앱 적용. 좌우 화살표, 키보드 ←/→, 드래그로 넘기고, 두 뷰가 같은 시간축으로 동기화되어 무한 루프로 재생됩니다.
- **한 사이클 모션**: 나타남(납작) → L1 떠오름·부유 → L2 확산·안착 + 리스트 생성 → L3 전체 화면 글로우 → 상단으로 수축하며 사라짐 → 반복
- **Design 탭**
  - 컬러: 3스톱 그라데이션 에디터(칩 드래그로 위치 조절), 레퍼런스 영상에서 추출한 프리셋 3종(Azure / Blossom / Indigo), 방사형·선형
  - 물성 5종: 소프트 글로우, 프로스티드 글래스, 리퀴드 메탈, 펄, 매트 클레이 (타입별 세부 파라미터)
  - 크기: 구 지름, 납작 상태 폭·두께, 안착 높이·두께·모서리, 여백
  - 모션: 재생 속도, 전환 시간, 레벨별 유지 시간, 전환 방식(모프 / 위상 회전), 이징 프리셋, 부유, 그림자
- **Code 탭**: 현재 설정이 그대로 들어간 Kotlin 파일, AGSL 셰이더, 설정 JSON을 보여주고 복사할 수 있습니다.
- 다크/라이트 모드, 설정 자동 저장(브라우저 localStorage)

## 폴더 구조

```
.
├─ index.html                 # 툴 전체 (단일 파일, GitHub Pages 진입점)
├─ android/
│  └─ AiVisualCue.kt          # 기본 설정으로 생성한 Compose 코드 예시
├─ docs/
│  ├─ development-plan.md     # Phase별 개발 계획서
│  └─ prompt.md               # 최초 요구사항 프롬프트
├─ .nojekyll
└─ README.md
```

## 구조: 하나의 상태에서 프리뷰와 코드를 함께 만든다

```
          ┌──────────── Spec (컨트롤 값 전체, JSON) ────────────┐
          │                                                      │
  타임라인 · 포즈 계산                                  코드 생성기
          │                                                      │
  WebGL2 셰이더 (GLSL ES 3.0) ◀── 같은 셰이더 소스 ──▶ AGSL (RuntimeShader)
          │                                                      │
     웹 프리뷰                                     Kotlin + Jetpack Compose
```

- 셰이더 본문은 `float2/float3/float4` 공통 문법으로 작성되어 있고, 웹에서는 매크로로 GLSL 타입에 연결하며 Android에서는 그대로 AGSL이 됩니다.
- 형태(납작 → 구 → 카드 → 전체 화면)는 둥근 사각형 SDF 하나로 표현하고, 레벨 사이는 파라미터 보간으로 연속 변형됩니다.
- 이징은 `cubic-bezier` 4개 숫자를 웹과 Compose `CubicBezierEasing`에 똑같이 사용합니다. 기본값은 레퍼런스 영상 리듬(약 0.3초 가속 → 0.4초 감속)에 맞춘 `0.6, 0, 0.1, 1`입니다.

## 생성 코드 사용법 (Android)

1. 툴의 Code 탭에서 **복사**를 눌러 `AiVisualCue.kt`로 프로젝트에 붙여넣습니다. 패키지명은 프로젝트에 맞게 바꿉니다.
2. 화면에서 호출합니다.

```kotlin
// 툴과 같은 무한 루프
AiVisualCueDemoLoop()

// 앱 상태로 레벨 제어
AiVisualCue(level = CueLevel.L2)
```

| 항목 | 내용 |
|---|---|
| minSdk | 26 |
| Android 13 (API 33)+ | AGSL `RuntimeShader`로 프리뷰와 같은 셰이더 사용 |
| API 26–32 | 그라데이션 `Brush`와 그림자로 근사하는 대체 경로 |
| 의존성 | Compose BOM 기본 라이브러리(ui, foundation, animation)만 사용 |

> 생성 코드는 아직 실제 Android 기기에서 컴파일·실행 검증을 거치지 않았습니다. 빌드 오류가 나면 이슈로 남겨 주세요.

## GitHub Pages 설정

`Settings → Pages → Build and deployment`에서 **Source: Deploy from a branch**, **Branch: `main` / `(root)`**를 선택하면 위 링크로 배포됩니다.

## 참고

- 기본 팔레트 3종과 모션 리듬은 레퍼런스 영상(field_io "Quantum / Qubit" 포스트)에서 측정한 값을 사용했습니다.
- 물성 5종, 레벨별 배치, 앱 목업 화면은 v0.1의 임시 정의입니다. 자세한 가정과 이후 단계는 [개발 계획서](docs/development-plan.md)를 참고하세요.

---
name: figma
description: Figma 디자인을 Atomic CSS 클래스만으로 HTML로 변환. Figma URL이나 디자인 구현 요청 시 사용. 인라인 스타일 금지, CSS 파일 금지 — 오직 Atomic CSS 클래스만 사용.
argument-hint: "[Figma URL 또는 구현할 디자인 설명]"
---

# Figma → Atomic CSS 변환기

> Figma 디자인 데이터를 읽어서 **100% Atomic CSS 클래스만으로** HTML을 생성합니다.
> 인라인 스타일 없이, CSS 파일 없이, 오직 클래스만.

## 절대 규칙

1. **Atomic CSS 클래스만 사용** — `style` 속성 금지, `<style>` 태그 금지, CSS 파일 금지
2. **Figma 색상 그대로** — HEX 값 절대 변경 금지, 반올림 금지 (Figma `#4A90D9` → `bg4A90D9`)
3. **간격은 4의 배수로 스냅** — Figma 13px → `12px`, 15px → `16px` (가장 가까운 4의 배수)
4. **클래스를 추측하지 말 것** — 확신 없으면 `atomic-css` MCP `lookup_class`로 반드시 검증
5. **모든 요소 변환 필수** — Figma에 보이는 요소를 임의로 생략하지 말 것. 탭, 네비게이션, 뱃지, 구분선, 상태 표시 등 모든 UI 요소를 빠짐없이 HTML로 변환해야 함
6. **높이 채움 필수** — 콘텐츠 영역, 테이블, 패널 등이 부모 높이를 채워야 할 경우 반드시 높이 채움 처리 (h100p, fg1, minh100vh 등)
7. **의미 없는 빈 `<div>` 래퍼 금지** — 스타일도 없고 시맨틱 의미도 없는 `<div>`로 감싸지 말 것. 최상위 루트가 필요하면 시맨틱 태그(`<section>`, `<article>`, `<main>` 등)를 사용하거나, 페이지 식별용 클래스를 부여할 것 (예: `class="page-bump-acr-error"`). 구분용 클래스는 스타일 정의가 아닌 **페이지/섹션의 명칭**으로만 사용
8. **구분선(Divider) 변환 필수** — Figma의 `<line>`, `<rectangle>`(얇은 직사각형), Separator 등 시각적 구분 요소는 `<hr>` 또는 `border-left`/`border-right`/`border-top`/`border-bottom`으로 반드시 변환. "장식적이라 생략" 금지

---

## Step 1: Figma 데이터 수집 (분할 전략)

큰 페이지도 자동 분할로 처리 가능.

### 절대 금지
- **추론으로 HTML 작성 금지** — MCP 검증 없이 "아마 이 클래스일 것이다"로 작성하면 안 됨
- **일반 CSS 클래스 사용 금지** — `class="header"`, `class="card-title"` 같은 시맨틱 클래스명은 Atomic CSS가 아님
- **`<style>` 태그나 CSS 파일 생성 금지** — 어떤 상황에서도 별도 CSS를 만들지 않음
- **Figma 데이터가 불완전해도 추측 금지** — 누락된 정보는 사용자에게 질문

### Figma MCP가 HTML+CSS 코드를 반환하는 경우 (우선 경로)

Figma MCP(`get_design_context` 등)가 HTML+CSS 코드를 반환하면, CSS 속성을 직접 추출할 수 있어 더 정확합니다:

1. **HTML+CSS 코드 수신** — Figma MCP에서 코드 출력을 받음
2. **CSS 속성 추출** — 코드에서 사용된 CSS 선언(color, padding, display 등)을 수집
3. **`css_to_classes`로 Atomic 변환** — 추출한 CSS를 Atomic CSS 클래스로 변환
4. **HTML 재구성** — Atomic 클래스로 교체한 HTML을 생성
5. **`validate_classes`로 검증** — 최종 검증 후 출력

> Figma MCP가 코드를 반환하지 않고 메타데이터만 반환하는 경우, 아래 기본 분할 전략으로 폴백합니다.

### 기본 분할 전략 (메타데이터만 반환된 경우)

**1-1. 최초 호출** — 전체 페이지 `get_design_context` 호출
```
→ 상세 데이터가 오면 → Step 2로
→ "too large" 메타데이터만 오면 → 1-2로
```

**1-2. 구조 분석 + 사용자 선택** — 메타데이터에서 직계 자식 frame ID를 추출하고, **번호 목록으로 사용자에게 제시**하여 필요한 섹션만 선택받는다.
```
컨텍스트가 큽니다. 어떤 섹션이 필요하세요?
1. header (1920x60)
2. sidebar (280x900)
3. main content (1640x900)
4. footer (1920x80)
→ 번호로 선택해주세요 (예: 3,4)
```
> **모든 섹션을 자동 호출하지 말 것.** 사용자가 선택한 섹션만 호출한다.
> header, sidebar 등은 이미 프로젝트에 존재할 수 있으므로 불필요한 호출을 줄인다.

**1-3. 선택된 섹션만 상세 호출** — 사용자가 선택한 섹션 ID로만 `get_design_context` 재호출
```
→ 색상, 폰트, padding, gap, border 등 상세 스타일 데이터가 전부 옴
→ 여전히 크면 그 안의 자식으로 재귀 분할
```

**1-4. 전체 조합** — 스크린샷을 보고 시맨틱 구조 판단 후 합침
```html
<div class="dg gtrauto-1fr-auto minh100vh">
    <header class="..."><!-- 섹션 ② --></header>
    <div class="dg gtc28rem-1fr">
        <aside class="..."><!-- 섹션 ③ --></aside>
        <main class="..."><!-- 섹션 ④ --></main>
    </div>
    <footer class="..."><!-- 섹션 ⑤ --></footer>
</div>
```

**1-5. 요소 완전성 체크** — 조합 후 Figma 원본과 대조하여 누락 요소 확인
```
체크리스트:
□ 탭/네비게이션 바 — 모든 탭 아이템이 존재하는가?
□ 상태 표시 (뱃지, 태그, 상태 아이콘) — 빠진 것 없는가?
□ 구분선 (Divider/Separator) — Figma에 있는 구분선이 모두 있는가?
□ 배경색 — 각 섹션의 배경색이 Figma와 일치하는가?
□ 중첩 컴포넌트 — instance 안의 자식 요소가 모두 변환되었는가?
```
> **원칙: Figma에 눈에 보이는 요소는 반드시 HTML로 변환한다. "불필요해 보인다"는 판단으로 생략하지 않는다.**

---

## Step 2: Figma 속성 → Atomic CSS 매핑

### 크기

| Figma 속성 | Atomic CSS | 규칙 |
|-----------|-----------|------|
| width: 200px (Fixed) | `w200px` 또는 `w20rem` | 20px 미만 → px, 20px 이상 → rem |
| height: 60px (Fixed) | `h6rem` | 동일 |
| Hug contents | 클래스 없음 | CSS 기본값 (내용에 맞춤) |
| Fill container | `w100p` 또는 `fg1` | 부모를 채움 |
| Min/Max width | `minw{값}px` / `maxw{값}px` | 있으면 추가 |

### 색상 (절대 변경 금지)

| Figma 속성 | Atomic CSS | 예시 |
|-----------|-----------|------|
| fill color | `bg{HEX}` | `#4A90D9` → `bg4A90D9` |
| text color | `c{HEX}` | `#333333` → `c333333` |
| stroke color | `bc{HEX}` 또는 border shorthand | `#DDDDDD` → `bcDDDDDD` |
| rgba fill | `bg{R}-{G}-{B}-{A}` | rgba(0,0,0,0.5) → `bg0-0-0-50` |
| opacity | `o{값}` | 0.5 → `o50` |
| gradient (linear) | `bglg-to-r-{HEX1}-{HEX2}` | 방향 + 색상 |
| gradient (radial) | `bgrg-circle-{HEX1}-{HEX2}` | |

### 타이포그래피

| Figma 속성 | Atomic CSS | 규칙 |
|-----------|-----------|------|
| fontSize | `fs{값}px` | **Figma 값 그대로** (스냅 안 함) |
| fontWeight | `fw{값}` | Regular=400, Medium=500, SemiBold=600, Bold=700 |
| lineHeight | `lh{값}px` 또는 `lh{비율}` | Figma 값 그대로 |
| letterSpacing | `ls{값}px` | 음수면 `neg-ls{값}px` |
| textAlign | `tac` / `tal` / `tar` | center / left / right |

```html
<p class="fs16px fw700 c333333 lh24px">제목</p>
<span class="fs14px fw400 c666666 neg-ls0-5px">보조 텍스트</span>
```

### 테두리

| Figma 속성 | Atomic CSS | 규칙 |
|-----------|-----------|------|
| stroke + weight | `b1pxsolid{HEX}` | border shorthand |
| 방향별 stroke | `bt1pxsolid{HEX}` | top만 있으면 `bt` |
| border-radius | `br{값}px` | 원형이면 `br50p` |
| 개별 corner radius | `btlr8px` `btrr8px` `bblr8px` `bbrr8px` | 각 꼭짓점 |
| stroke inside | border shorthand + `bxzbb` | box-sizing: border-box |
| stroke outside | border shorthand 그대로 | CSS 기본 동작 |

### 간격 (padding, gap)

| Figma 속성 | Atomic CSS | 규칙 |
|-----------|-----------|------|
| padding 동일 | `p{값}px` | 4의 배수로 스냅 |
| padding 상하/좌우 다름 | `pt{값}px pr{값}px pb{값}px pl{값}px` | 개별 지정 |
| padding 상하 같고 좌우 같음 | `p{상하}px-{좌우}px` | shorthand 2값 |
| Auto Layout gap | `gap{값}px` | 4의 배수로 스냅 |

```html
<!-- Figma: padding top 20, right 16, bottom 20, left 16 -->
<div class="pt2rem pr16px pb2rem pl16px">

<!-- 또는 상하/좌우 같으면 -->
<div class="p2rem-16px">
```

### 섹션 간 간격 (형제 요소 사이 여백)

> **Figma에서 형제 프레임/컴포넌트 사이에 간격이 있으면 반드시 반영할 것.**
> 이 간격을 누락하면 요소가 붙어버려 디자인이 깨진다.

**간격 확인 방법:**
1. 부모 Auto Layout에 `gap` 값이 있으면 → 부모에 `gap{값}px`
2. 부모에 gap이 없고 개별 프레임에 margin이 있으면 → 해당 요소에 `mt{값}px` 등
3. Figma에서 두 프레임 사이 거리를 직접 측정 → 가장 가까운 4의 배수로 스냅

**특히 놓치기 쉬운 케이스:**
| 위치 | 설명 | 예시 |
|------|------|------|
| ContentBar ↔ Table | 버튼 영역과 테이블 사이 | `mt8px` 또는 부모 `gap8px` |
| Header ↔ Content | 헤더와 본문 사이 | 부모 `gap16px` |
| Tab ↔ TabPanel | 탭 바와 내용 영역 사이 | `mt12px` 또는 부모 `gap12px` |
| Form 필드 사이 | 라벨+인풋 그룹 간 | 부모 `gap16px` |
| 카드 ↔ 카드 | 카드 목록 간격 | 부모 `gap16px` |

> **원칙: 부모에 gap이 있으면 gap으로, 없으면 개별 margin으로 처리. 간격 0px이 아닌 이상 절대 생략하지 않는다.**

### 기타

| Figma 속성 | Atomic CSS |
|-----------|-----------|
| overflow hidden (clip) | `oh` |
| position absolute | `pa` + `t{값}px` + `l{값}px` (부모에 `pr`) |
| box-shadow | `bs{x}px{y}px{blur}px{spread}px{HEX}` |
| backdrop blur | `bfb{값}px` |

---

## Step 3: 레이아웃 변환

### Grid 기본 원칙

**모든 레이아웃은 Grid(`dg` + `gtc`/`gtr`)를 기본으로 사용.**
Flex는 아이콘+텍스트 같은 소규모 인라인 정렬에만 보조적으로 사용.

| Figma 패턴 | 변환 |
|-----------|------|
| 페이지 구조 (헤더+메인+푸터) | `dg gtrauto-1fr-auto` |
| 사이드바 + 콘텐츠 | `dg gtc28rem-1fr` |
| 카드 그리드 (반복) | `dg gtcr3-1fr` 또는 `gtcrfit-minmax28rem-1fr` |
| 수평 나열 (메뉴, 탭) | `dg gtcauto-auto-auto gap8px` |
| 양쪽 정렬 (로고 ↔ 메뉴) | `dg gtc1fr-auto` |
| 폼 필드 (라벨+인풋) | `dg gap4px` |
| 버튼 안 아이콘+텍스트 | `df aic gap8px` (예외: Flex 허용) |

### Figma Auto Layout → Grid 변환

```html
<!-- Horizontal + gap 16 + 자식 3개 -->
<div class="dg gtcauto-auto-auto gap16px aic">

<!-- Horizontal + gap 16 + 자식이 공간을 균등 채움 -->
<div class="dg gtc1fr-1fr-1fr gap16px">

<!-- Vertical + gap 12 -->
<div class="dg gap12px">

<!-- Horizontal + space-between (양쪽 정렬) -->
<div class="dg gtc1fr-auto">

<!-- Horizontal + 고정 + 유동 -->
<div class="dg gtc200px-1fr gap16px">

<!-- 반복 카드 (auto-fit) -->
<div class="dg gtcrfit-minmax28rem-1fr gap16px">

<!-- 버튼 내부 아이콘+텍스트 (예외: Flex) -->
<button class="df aic gap8px">
```

### 높이 채움 패턴

콘텐츠 영역, 테이블, 패널 등이 **남은 공간을 채워야 하는 경우** 반드시 처리:

| 패턴 | 변환 | 설명 |
|------|------|------|
| 페이지 전체 높이 | `minh100vh` 또는 `h100vh` | 뷰포트 전체 채움 |
| Grid 자식이 남은 높이 채움 | `gtr` 에서 `1fr` 사용 | `gtrauto-1fr-auto` |
| Flex 자식이 남은 높이 채움 | `fg1` + `minh0` | flex-grow로 채움 |
| 테이블 래퍼가 남은 높이 채움 | `fg1 oh` 또는 `h100p oh` | 스크롤 가능 영역 |
| 중첩 컨테이너 높이 전파 | 부모~자식 모두 `h100p` 체인 | 높이가 끊기지 않도록 |

```html
<!-- 전형적 대시보드: 헤더 + 탭 + 테이블이 나머지 채움 -->
<div class="dg gtrauto-auto-1fr h100vh">
    <header class="...">헤더</header>
    <nav class="df gap16px ...">탭 네비게이션</nav>
    <div class="oh">
        <table class="w100p">...</table>
    </div>
</div>

<!-- Flex 레이아웃에서 콘텐츠 채움 -->
<div class="df fdc h100vh">
    <header class="...">헤더</header>
    <main class="fg1 minh0 oya">콘텐츠 (스크롤)</main>
    <footer class="...">푸터</footer>
</div>
```

### 절대 좌표 자유 배치 → Grid 변환

Figma에서 Auto Layout 없이 **절대 좌표(x, y)로 자유 배치**된 자식 프레임들은 균일 Grid로 강제 변환하지 말 것.

**변환 절차:**
1. **스크린샷 기준으로 시각적 행/열 그룹 파악** — 각 프레임의 x, y 좌표를 분석
2. **동일 y 좌표 범위 → 같은 행**, 동일 x 좌표 범위 → 같은 열
3. **열 수가 행마다 다르면** 각 행을 별도 Grid로 구성
4. **각 프레임의 실제 너비를 `gtc` 값에 반영** — `gtcr5-auto` 같은 균일 반복 금지, 실제 크기 기반 `gtc{w1}px-{w2}px-...` 사용

```html
<!-- ❌ 크기가 다른 프레임을 균일 Grid로 강제 -->
<div class="dg gtcr5-auto gap8px">
    <div>STK-001 (200px)</div>
    <div>STK-002 (300px)</div>
    <div>STK-003 (150px)</div>
</div>

<!-- ✅ 실제 Figma 좌표/크기를 분석하여 반영 -->
<div class="dg gtc200px-300px-150px gap8px">
    <div>STK-001</div>
    <div>STK-002</div>
    <div>STK-003</div>
</div>
```

> **원칙: Figma 좌표를 무시하고 균일 그리드로 단순화하면 레이아웃 전체가 달라진다. 반드시 실제 크기를 반영할 것.**

### 절대 위치 요소 (오버레이/배지)

```html
<div class="pr">
    <div class="pa t10px l10px">오버레이</div>
    <div class="pa b0 r0">우하단 배지</div>
</div>
```

---

## Step 4: 간격 스냅 규칙

4의 배수가 아닌 Figma 값은 가장 가까운 4의 배수로 스냅:

| Figma | 스냅 | | Figma | 스냅 |
|-------|------|-|-------|------|
| 1~2px | `0` 또는 제거 | | 13px | `12px` |
| 3px | `4px` | | 14~15px | `16px` |
| 5px | `4px` | | 17px | `16px` |
| 6~7px | `8px` | | 18~19px | `2rem` (20px) |
| 9px | `8px` | | 21px | `2rem` (20px) |
| 10~11px | `12px` | | 22~23px | `2-4rem` (24px) |

> 20px 이상은 rem으로 전환: 20px=`2rem`, 24px=`2-4rem`, 32px=`3-2rem`, 40px=`4rem`

**스냅하지 않는 속성** (Figma 값 그대로):
- font-size, border-radius, border-width, width/height (고정 크기), 색상

---

## Step 5: 이미지/아이콘 처리

| 유형 | 처리 |
|------|------|
| SVG 아이콘 (데이터 있음) | `<svg>` 직접 삽입, Figma 크기/색상 그대로 |
| 아이콘 (SVG 없음) | `<span class="dib w2-4rem h2-4rem"></span>` 빈 영역 |
| 사진/이미지 | `<div class="w100p h20rem bgF0F0F0 br8px"></div>` placeholder |
| 아바타 (원형) | `<div class="w4rem h4rem br50p bgDDDDDD"></div>` |
| 배경 이미지 | `<div class="w100p ar16-9 bgE0E0E0 br8px"></div>` |

---

## Step 6: 반응형 대응 (필수)

> **모든 고정 레이아웃에 반드시 반응형 클래스를 추가할 것.**
> 고정 px/rem만 사용하고 반응형 미적용은 금지.

### 반응형 적용 필수 대상

| 대상 | 필수 처리 | 예시 |
|------|----------|------|
| Grid 다중 컬럼 | `md-`, `sm-` 컬럼 축소 | `gtcr3-1fr md-gtcr2-1fr sm-gtc1fr` |
| 사이드바/패널 | 태블릿 축소, 모바일 숨김 | `w28rem md-w24rem sm-dn` |
| 고정 너비 컨테이너 | `maxw` + `w100p` 조합 | `maxw120rem w100p` |
| 큰 폰트 (2rem 이상) | 단계별 축소 | `fs2-4rem md-fs2rem sm-fs1-8rem` |
| 고정 높이 레이아웃 | 모바일에서 자동 높이 | `h100vh sm-ha` |
| 좌우 padding (2rem 이상) | 모바일 축소 | `p3-2rem sm-p16px` |
| 고정 gap (2rem 이상) | 모바일 축소 | `gap2-4rem sm-gap16px` |
| 가로 나열 요소 (5개+) | 모바일에서 스크롤 또는 축소 | `oxa` 또는 컬럼 변경 |

### 반응형 판단 기준

```
너비 고정값 사용? → md-/sm- 축소 또는 100% 전환 필수
다중 컬럼 Grid? → sm-에서 1컬럼으로 축소 필수
사이드바 존재? → sm-에서 숨김 또는 접기 필수
큰 여백/폰트? → sm-에서 축소 필수
테이블? → sm-에서 oxa (가로 스크롤) 추가
```

### 예시

```html
<!-- 그리드 컬럼: 3열 → 2열 → 1열 -->
<div class="dg gtcr3-1fr md-gtcr2-1fr sm-gtc1fr gap16px">

<!-- 사이드바: 데스크탑 보임 → 모바일 숨김 -->
<aside class="w28rem md-w24rem sm-dn">

<!-- 폰트 크기 -->
<h1 class="fs2-4rem md-fs2rem sm-fs1-8rem">

<!-- 테이블: 모바일 가로 스크롤 -->
<div class="sm-oxa">
    <table class="w100p sm-minw60rem">...</table>
</div>

<!-- 고정 너비 → 유동 -->
<div class="w60rem md-w100p p2-4rem sm-p16px">
```

---

## Step 7: 컴포넌트 인스턴스 처리

Figma `<instance>`를 만나면:
1. 해당 인스턴스 ID로 `get_design_context` 개별 호출 → 내부 구조 확인
2. 반복되는 인스턴스는 동일 HTML 구조로 생성
3. 인스턴스 이름이 역할을 나타내면 참고 (예: "Button/Primary", "Card/Default")

---

## Step 8: 작업 대상 파일 확인

사용자가 Figma URL만 주고 **어디에 적용할지 명시하지 않은 경우**, 반드시 확인:

1. **프로젝트 구조 탐색** — Figma 페이지명, 컴포넌트명 등 힌트로 관련 파일을 검색하여 후보 목록 작성
2. **번호 목록으로 제시** — 후보 파일과 "새 파일 생성" 옵션을 번호로 나열
3. **절대 추측하지 말 것** — 파일 경로를 임의로 결정하여 작업하지 않음

```
사용자: "이 피그마대로 만들어줘" (URL만 제공)
→ ❌ 임의로 index.html 생성
→ ✅ 후보를 번호로 제시:
   어떤 파일에 적용할까요?
   1. stoker-map-status.vue
   2. stoker-status.vue
   3. 새 파일 생성
   → 번호로 선택해주세요:
```

> 단, 사용자가 이전 대화에서 파일을 이미 언급했거나, 현재 열린 파일이 명확한 경우는 추가 질문 없이 진행.

---

## Step 9: 기존 코드가 있는 파일에 적용

대상 파일에 **이미 코드가 존재하는 경우**, Figma 결과물로 무조건 덮어쓰지 말 것.

### 반드시 보존해야 할 것

| 보존 대상 | 예시 |
|----------|------|
| 이벤트 핸들러 | `onclick`, `@click`, `addEventListener` |
| 데이터 바인딩 | `v-for`, `v-if`, `{{data}}`, `{items.map(...)}` |
| 동적 클래스 | `:class`, `className={...}`, 조건부 클래스 |
| 컴포넌트 구조 | `<Component />`, import 문, props |
| 상태 관리 | `useState`, `ref()`, `computed` |
| ID/JS 셀렉터 | `id="..."`, `data-*`, `ref="..."` |

### 기존 컴포넌트 재활용 시 스타일 차이 반영

기존 프로젝트의 컴포넌트(`ContentBar`, `PageHeader`, `Sidebar` 등)를 재사용할 때, **Figma 스타일과 컴포넌트 기본 스타일이 다를 수 있다.** 차이를 무시하면 배경색, 패딩, 보더 등이 누락된다.

**반드시 확인할 항목:**
| 항목 | 확인 방법 |
|------|----------|
| 배경색 | Figma fill vs 컴포넌트 기본 배경 → 다르면 래퍼에 `bg{HEX}` 추가 |
| 패딩 | Figma padding vs 컴포넌트 기본 패딩 → 다르면 클래스 오버라이드 |
| 보더 | Figma stroke vs 컴포넌트 기본 보더 → 다르면 추가 |
| 간격(gap) | Figma gap vs 컴포넌트 기본 gap → 다르면 추가 |

```html
<!-- Figma: 필터 바 배경 #EBF3FA, 컴포넌트 기본은 투명 -->
<!-- ❌ 컴포넌트 그대로 사용 → 배경색 누락 -->
<ContentBar>...</ContentBar>

<!-- ✅ Figma 고유 스타일을 클래스로 보완 -->
<ContentBar class="bgEBF3FA">...</ContentBar>
```

### 작업 절차

1. **기존 파일 먼저 읽기** — 현재 구조, 로직, 바인딩 파악
2. **기존 컴포넌트 탐색** — Figma의 각 섹션(header, filter bar, sidebar 등)에 대응하는 기존 컴포넌트가 `app/components/`에 있는지 탐색. 있으면 재사용, 없으면 새로 마크업
3. **Figma와 기존 코드 비교** — 어떤 부분이 변경되고, 어떤 부분이 유지되는지 판단. 재사용 컴포넌트는 기본 스타일과 Figma 스타일 차이를 확인
4. **스타일(클래스)만 교체** — 기존 로직은 유지하고, Atomic CSS 클래스만 피그마 기준으로 갱신
5. **새 요소 추가** — Figma에는 있지만 기존 코드에 없는 요소는 적절한 위치에 삽입
6. **기존 요소 제거 금지** — Figma에 없더라도 로직/기능이 있는 요소는 함부로 삭제하지 않음. 사용자에게 확인 후 처리

### Figma에 없는 요소를 만들어야 할 때

사용자가 요청한 요소가 현재 피그마 화면에 존재하지 않는 경우:

1. **가이드 페이지 탐색** — 같은 Figma 파일 내 Guide, Design System, Components 등의 페이지에서 해당 요소와 가장 유사한 컴포넌트를 찾아 그 스타일을 적용
   - `get_design_context`로 가이드 페이지를 호출하여 색상, 간격, 보더, 폰트 등 확인
   - 버튼, 인풋, 모달, 토스트 등 공통 컴포넌트가 정의되어 있을 가능성 높음

2. **가이드에도 없는 경우** — 현재 피그마 디자인에서 사용된 패턴을 추출하여 일관되게 적용
   - **색상**: 이미 사용된 배경색, 텍스트 색상, 보더 색상을 재사용
   - **간격**: 기존 디자인의 padding, gap 패턴 유지 (예: 카드가 `p16px gap12px`이면 새 요소도 동일)
   - **타이포그래피**: 기존 제목/본문/보조 텍스트의 fs, fw, c 패턴 따름
   - **보더/라운드**: 기존 카드, 버튼의 br, border 패턴 유지
   - **레이아웃**: 기존 디자인의 Grid/Flex 패턴과 동일한 방식 사용

```html
<!-- 기존 피그마 디자인의 카드 패턴 -->
<div class="p16px bgFFFFFF br8px b1pxsolidE0E0E0">
    <h3 class="fs16px fw600 c333333">제목</h3>
    <p class="fs14px fw400 c666666">내용</p>
</div>

<!-- ✅ 피그마에 없는 "알림 카드"를 같은 패턴으로 생성 -->
<div class="p16px bgFFF8E1 br8px b1pxsolidFFCC02">
    <h3 class="fs16px fw600 c333333">알림 제목</h3>
    <p class="fs14px fw400 c666666">알림 내용</p>
</div>
```

```html
<!-- 기존 코드 -->
<div class="old-class" v-for="item in items" :key="item.id">
    <span @click="handleClick(item)">{{ item.name }}</span>
</div>

<!-- ✅ 올바른 적용: 클래스만 교체, 로직 보존 -->
<div class="dg gap12px p16px bgFFFFFF br8px" v-for="item in items" :key="item.id">
    <span class="fs14px fw500 c333333 cp" @click="handleClick(item)">{{ item.name }}</span>
</div>

<!-- ❌ 잘못된 적용: 로직 제거하고 정적 HTML만 생성 -->
<div class="dg gap12px p16px bgFFFFFF br8px">
    <span class="fs14px fw500 c333333">상품명</span>
</div>
```

---

## 사용자 요청

$ARGUMENTS

## 생성 가이드라인

1. **Atomic CSS 클래스만** — 인라인 스타일, CSS 파일 절대 금지
2. **Figma 색상 1글자도 바꾸지 말 것**
3. **Grid/Flex 판단 기준** 따를 것 (Step 3 참고)
4. **간격 4의 배수 스냅** (font-size, border-radius 제외)
5. **시맨틱 HTML** — header, nav, main, section, aside, footer 적극 사용. 의미 없는 빈 `<div>` 래퍼 금지. 루트 요소가 필요하면 시맨틱 태그 또는 페이지 식별 클래스 사용
6. **이미지/아이콘** — SVG 있으면 삽입, 없으면 placeholder
7. **반응형** — 주요 레이아웃에 `md-`, `sm-` 변형 추가

## 생성 후 검증 (필수)

HTML 생성 후, 사용자에게 제시하기 전에 반드시 수행:

### A. 기본 검증
1. **클래스 전수 검증** — `atomic-css` MCP `validate_classes`로 모든 클래스 확인. 무효 클래스 수정
2. **인라인 스타일 0개** — `style=` 속성 없음 확인
3. **CSS 파일 0개** — `<style>` 태그, 외부 CSS 없음 확인
4. **간격 4의 배수** — padding, margin, gap 값 확인 (font-size 제외)
5. **단위 규칙** — 20px 미만 → px, 20px 이상 → rem
6. **HEX 대문자** — 모든 색상 6자리 대문자

### B. 요소 완전성 검증 (누락 방지)
7. **Figma 요소 1:1 대조** — Figma 데이터의 모든 자식 노드가 HTML에 존재하는지 확인
   - 탭/네비게이션 바의 모든 아이템
   - 뱃지, 태그, 상태 아이콘
   - 구분선 (Divider, Separator, Line)
   - 빈 공간 placeholder
   - 비활성/숨김 상태 요소도 포함 (opacity, disabled 등으로 표현)
8. **누락 발견 시** — Figma 데이터를 다시 확인하고 빠진 요소를 추가

### C. 색상 정확성 검증
9. **배경색 전수 비교** — Figma 각 프레임/컴포넌트의 fill 색상과 HTML `bg{HEX}` 클래스가 정확히 일치하는지 확인
   - 특히 유사 색상 (예: `bgFFFFFF` vs `bgF5F5F5`, `bgFAFAFA`) 혼동 주의
   - 각 섹션/영역의 배경색을 Figma 데이터에서 개별 확인
10. **텍스트 색상** — Figma text fill과 `c{HEX}` 일치 확인
11. **보더 색상** — Figma stroke와 `bc{HEX}` 또는 border shorthand 색상 일치 확인

### D. 레이아웃 검증
12. **높이 채움** — 콘텐츠 영역, 테이블, 패널이 남은 공간을 채우는지 확인
    - `h100vh`, `1fr`, `fg1`, `h100p` 등이 적절히 사용되었는지
    - 높이가 끊기는 구간이 없는지 (부모~자식 체인 확인)
13. **섹션 간 간격** — 형제 요소(Header↔Content, ContentBar↔Table, Tab↔Panel 등) 사이 간격이 Figma와 일치하는지 확인. gap 또는 margin으로 처리되어 있어야 함
14. **레이아웃 구조** — Figma 스크린샷과 비교하여 구조 일치 확인
15. **반응형 필수** — 모든 고정 레이아웃에 `md-`, `sm-` 변형 존재 확인
    - Grid 다중 컬럼 → 축소 처리 확인
    - 사이드바/패널 → 모바일 처리 확인
    - 테이블 → 가로 스크롤 처리 확인

## Atomic CSS 네이밍 참고

**CSS 속성 + 값의 앞글자 조합:**
```
display: flex → df     justify-content: center → jcc
align-items: center → aic     flex-direction: column → fdc
```

**모르는 클래스는 반드시 MCP로 조회:**
- `lookup_class` — 클래스 → CSS
- `search_by_css` — CSS 속성명으로 검색
- `css_to_classes` — CSS → Atomic 클래스
- `validate_classes` — 유효성 검증

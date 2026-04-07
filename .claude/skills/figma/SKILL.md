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

---

## Step 1: Figma 데이터 수집 (분할 전략)

큰 페이지도 자동 분할로 처리 가능.

**1-1. 최초 호출** — 전체 페이지 `get_design_context` 호출
```
→ 상세 데이터가 오면 → Step 2로
→ "too large" 메타데이터만 오면 → 1-2로
```

**1-2. 구조 분석** — 메타데이터에서 직계 자식 frame ID 추출
```
1920x1080 페이지 예시:
├── header (1920x60)       ← 개별 호출
├── sidebar (280x900)      ← 개별 호출
├── main content (1640x900) ← 크면 재분할
└── footer (1920x80)       ← 개별 호출
```

**1-3. 섹션별 상세 호출** — 각 섹션 ID로 `get_design_context` 재호출
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

### 절대 위치 요소

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

## Step 6: 반응형 대응

Figma 디자인이 데스크탑 기준이면, 핵심 레이아웃에 모바일/태블릿 대응 추가:

```html
<!-- 그리드 컬럼: 3열 → 2열 → 1열 -->
<div class="dg gtcr3-1fr md-gtcr2-1fr sm-gtc1fr gap16px">

<!-- 사이드바: 데스크탑 보임 → 모바일 숨김 -->
<aside class="w28rem md-w24rem sm-dn">

<!-- 폰트 크기 -->
<h1 class="fs2-4rem md-fs2rem sm-fs1-8rem">
```

---

## Step 7: 컴포넌트 인스턴스 처리

Figma `<instance>`를 만나면:
1. 해당 인스턴스 ID로 `get_design_context` 개별 호출 → 내부 구조 확인
2. 반복되는 인스턴스는 동일 HTML 구조로 생성
3. 인스턴스 이름이 역할을 나타내면 참고 (예: "Button/Primary", "Card/Default")

---

## 사용자 요청

$ARGUMENTS

## 생성 가이드라인

1. **Atomic CSS 클래스만** — 인라인 스타일, CSS 파일 절대 금지
2. **Figma 색상 1글자도 바꾸지 말 것**
3. **Grid/Flex 판단 기준** 따를 것 (Step 3 참고)
4. **간격 4의 배수 스냅** (font-size, border-radius 제외)
5. **시맨틱 HTML** — header, nav, main, section, aside, footer 적극 사용
6. **이미지/아이콘** — SVG 있으면 삽입, 없으면 placeholder
7. **반응형** — 주요 레이아웃에 `md-`, `sm-` 변형 추가

## 생성 후 검증 (필수)

HTML 생성 후, 사용자에게 제시하기 전에 반드시 수행:

1. **클래스 전수 검증** — `atomic-css` MCP `validate_classes`로 모든 클래스 확인. 무효 클래스 수정
2. **색상 대조** — Figma HEX 값과 출력 클래스의 HEX가 정확히 일치하는지 1:1 대조
3. **인라인 스타일 0개** — `style=` 속성 없음 확인
4. **CSS 파일 0개** — `<style>` 태그, 외부 CSS 없음 확인
5. **간격 4의 배수** — padding, margin, gap 값 확인 (font-size 제외)
6. **단위 규칙** — 20px 미만 → px, 20px 이상 → rem
7. **HEX 대문자** — 모든 색상 6자리 대문자
8. **레이아웃** — Figma 스크린샷과 비교하여 구조 일치 확인
9. **반응형** — 주요 레이아웃에 `md-`, `sm-` 변형 존재 확인

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

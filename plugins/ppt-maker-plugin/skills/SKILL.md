# 순수 HTML/CSS/JS 웹 슬라이드 프레젠테이션 — 통합 리서치

> 외부 라이브러리 없이 구현하는 프로덕션급 슬라이드 엔진
> 4개 주제를 병렬 리서치한 결과를 통합 정리

---

## 목차

1. [추천 아키텍처](#1-추천-아키텍처)
2. [Agent 1 · 슬라이드 레이아웃 & DOM 구조](#2-agent-1--슬라이드-레이아웃--dom-구조)
3. [Agent 2 · 키보드 & 마우스 네비게이션](#3-agent-2--키보드--마우스-네비게이션)
4. [Agent 3 · 슬라이드 전환 애니메이션](#4-agent-3--슬라이드-전환-애니메이션)
5. [Agent 4 · 슬라이드 내 콘텐츠 애니메이션 (빌드 효과)](#5-agent-4--슬라이드-내-콘텐츠-애니메이션-빌드-효과)
6. [핵심 패턴 요약](#6-핵심-패턴-요약)
7. [주의사항 & 브라우저 호환성](#7-주의사항--브라우저-호환성)
8. [적용된 수정사항 (vanilla-slides.html)](#8-적용된-수정사항-vanilla-slideshtml)

---

## 1. 추천 아키텍처

```
레이아웃:    position: fixed + inset: 0 (풀뷰포트, 여백 없음)
콘텐츠 폭:   .slide-inner max-width: 680px + clamp() 반응형 크기
전환 효과:   Web Animations API (WAAPI) + Promise.all 병렬 처리
텍스트 크기: clamp(min, vw, max) — JS 없이 CSS만으로 반응형
GPU 가속:    will-change 전환 직전 적용, 완료 후 'auto' 해제
입력 차단:   #busy 플래그 (isAnimating)
Fragment:    슬라이드 진입 시 자동 스태거 등장 (.fragment → .visible)
폰트:        Noto Sans KR (한글) + Inter (영문) + word-break: keep-all
```

### 최종 채택 HTML 골격

```html
<main class="presentation">           <!-- position: fixed; inset: 0 -->
  <div class="slide-viewport">        <!-- position: absolute; inset: 0 -->
    <section class="slide" data-state="current">
      <div class="slide-inner">       <!-- max-width: 680px 콘텐츠 폭 제한 -->
        ...
      </div>
    </section>
    <section class="slide" data-state="hidden">...</section>
  </div>
</main>

<div class="progress-bar"><div class="progress-fill"></div></div>
<nav class="dot-nav" role="tablist">...</nav>
<div class="slide-counter">1 / 10</div>
<div id="srAnnouncer" aria-live="polite"></div>  <!-- 스크린리더 알림 -->
```

### 최종 채택 CSS 골격

```css
/* 풀뷰포트 — 여백 없음 */
.presentation {
  position: fixed;
  inset: 0;
  overflow: hidden;
  background: #000;
}

/* 슬라이드 스택 컨테이너 */
.slide-viewport {
  position: absolute;
  inset: 0;
  overflow: hidden;
}

/* 슬라이드 스택 */
.slide {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  visibility: hidden;
  will-change: auto;
}
.slide[data-state="current"]  { visibility: visible; z-index: 10; }
.slide[data-state="entering"] { visibility: visible; z-index: 11; }
.slide[data-state="leaving"]  { visibility: visible; z-index:  9; }

/* 콘텐츠 폭 제한 */
.slide-inner {
  width: 100%;
  max-width: 680px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

/* 반응형 폰트 — 680px 콘텐츠 폭 기준 */
.slide h1 { font-size: clamp(1.5rem, 3.8vw, 2.6rem); }
.slide h2 { font-size: clamp(1.15rem, 2.6vw, 1.9rem); }
.slide p, .slide li { font-size: clamp(0.8rem, 1.4vw, 1.05rem); }
```

### 최종 채택 JS 골격

```js
class Presentation {
  #cur = 0;
  #busy = false;    // isAnimating flag

  /* 방향키 한 번 = 슬라이드 한 장 이동 (fragment 스텝 없음) */
  next() {
    if (this.#busy) return;
    if (this.#cur < this.#total - 1) this.#transition(this.#cur + 1, 'next');
  }

  /* 슬라이드 진입 시 fragment 전부 스태거 등장 */
  #revealFragments(idx) {
    [...this.#slides[idx].querySelectorAll('.fragment')]
      .forEach((f, i) => setTimeout(() => f.classList.add('visible'), i * 90));
  }

  async #transition(target, dir) {
    this.#busy = true;
    this.#hideFragments(this.#cur);
    this.#hideFragments(target);
    // ... WAAPI 전환 ...
    this.#revealFragments(target);
    this.#busy = false;
  }
}
```

---

## 2. Agent 1 · 슬라이드 레이아웃 & DOM 구조

### 2-1. Full-Viewport 레이아웃 방법 비교

| 방식 | 장점 | 단점 | 추천 용도 |
|------|------|------|-----------|
| **Flexbox** | translateX 전환 자연스러움 | 슬라이드 수 많으면 총 너비 증가 | 단순 수평 전환 |
| **CSS Grid** | `place-items: center` 한 줄로 중앙 정렬 | 이동 애니메이션 구현 복잡 | fade/크로스페이드 |
| **Absolute** | 복잡한 전환 자유롭게 구현 | 기준점 실수 시 레이아웃 오류 | 다양한 전환 효과 |

#### Flexbox 방식

```css
.slideshow {
  display: flex;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}
.slide {
  flex: 0 0 100vw;
  height: 100vh;
}
/* 전환: .slideshow에 translateX 적용 */
```

#### CSS Grid 방식 (fade에 최적)

```css
.slideshow {
  display: grid;
  grid-template-columns: 1fr;
  grid-template-rows: 1fr;
  width: 100vw;
  height: 100vh;
}
.slide {
  grid-column: 1 / -1;
  grid-row: 1 / -1;
  display: grid;
  place-items: center;
  opacity: 0;
  visibility: hidden;
  transition: opacity 0.4s ease;
}
.slide.active { opacity: 1; visibility: visible; }
```

#### Absolute Positioning 방식 (복잡한 전환에 최적)

```css
.slideshow {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}
.slide {
  position: absolute;
  inset: 0;
  visibility: hidden;
}
.slide[data-state="current"]  { visibility: visible; z-index: 10; }
.slide[data-state="entering"] { visibility: visible; z-index: 11; }
.slide[data-state="leaving"]  { visibility: visible; z-index:  9; }
```

---

### 2-2. 최적 HTML 구조 (시맨틱 + ARIA)

```html
<main id="presentation"
      role="region"
      aria-label="슬라이드 프레젠테이션"
      aria-roledescription="presentation">

  <div id="slide-deck" aria-live="polite" aria-atomic="true">

    <section id="slide-1"
             class="slide active"
             role="group"
             aria-roledescription="slide"
             aria-label="1 / 5: 제목 슬라이드"
             aria-hidden="false">
      <div class="slide-content">
        <h1>제목</h1>
        <p class="subtitle">부제목</p>
      </div>
    </section>

    <section id="slide-2"
             class="slide"
             role="group"
             aria-roledescription="slide"
             aria-label="2 / 5: 개요"
             aria-hidden="true">
      ...
    </section>

  </div>

  <nav class="slide-controls" aria-label="슬라이드 탐색">
    <button id="btn-prev" aria-label="이전 슬라이드" disabled>←</button>
    <div class="dot-nav" role="tablist" aria-label="슬라이드 목록">
      <button role="tab" aria-selected="true" aria-controls="slide-1"></button>
      <button role="tab" aria-selected="false" aria-controls="slide-2"></button>
    </div>
    <button id="btn-next" aria-label="다음 슬라이드">→</button>
  </nav>

  <!-- 스크린리더 전용 상태 알림 -->
  <div aria-live="assertive" aria-atomic="true"
       style="position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(0,0,0,0)">
    슬라이드 1 / 5
  </div>

</main>
```

**ARIA 속성 정리**

| ARIA 속성 | 적용 대상 | 목적 |
|-----------|-----------|------|
| `role="region"` | 프레젠테이션 래퍼 | 주요 콘텐츠 영역 명시 |
| `aria-roledescription="presentation"` | 래퍼 | "프레젠테이션"으로 읽힘 |
| `role="group"` | 개별 슬라이드 | 관련 콘텐츠 묶음 |
| `aria-roledescription="slide"` | 개별 슬라이드 | "슬라이드"로 읽힘 |
| `aria-hidden="true"` | 비활성 슬라이드 | 스크린리더 노출 제외 |
| `aria-live="polite"` | 슬라이드 덱 | 변경 시 비방해적으로 알림 |

---

### 2-3. 16:9 비율 고정 방법 비교

#### 방법 1: `aspect-ratio` (현대 브라우저 권장)

```css
.slide-viewport {
  aspect-ratio: 16 / 9;
  /* 뷰포트 내에서 letterbox */
  width:  min(100vw, 100vh * (16/9));
  height: min(100vh, 100vw * (9/16));
  margin: auto;
}
```

#### 방법 2: padding-top trick (구형 브라우저 폴백)

```css
.slide-wrapper {
  position: relative;
  width: 100%;
  padding-top: 56.25%;   /* 9/16 = 56.25% */
  overflow: hidden;
}
.slide-wrapper .slide {
  position: absolute;
  inset: 0;
}
```

#### 방법 3: @supports 폴백 패턴

```css
.slide {
  /* 구형 폴백 */
  width: 100%;
  padding-top: 56.25%;
  position: relative;
}
@supports (aspect-ratio: 16/9) {
  .slide {
    padding-top: 0;
    aspect-ratio: 16 / 9;
    position: static;
  }
}
```

---

### 2-4. 텍스트 크기 스케일링 방법 비교

| 방법 | 구현 | 장점 | 단점 |
|------|------|------|------|
| `transform: scale()` | JS로 계산 주입 | 완벽한 비례 유지 | 레이아웃 공간 비반영 |
| `clamp() + vw` | CSS만으로 가능 | JS 불필요 | 요소별 개별 설정 |
| `zoom` | JS 권장 | 레이아웃 반영 | CPU 리플로우 발생 |

```css
/* clamp 방식 (권장) */
h1 { font-size: clamp(1.8rem, 5vw, 4rem); }
h2 { font-size: clamp(1.4rem, 3.5vw, 2.8rem); }
p  { font-size: clamp(0.9rem, 1.8vw, 1.4rem); }
```

```js
// transform scale 방식 (고정 캔버스 디자인 시)
function scaleSlide() {
  const BASE_W = 1920, BASE_H = 1080;
  const scale = Math.min(innerWidth / BASE_W, innerHeight / BASE_H);
  const offsetX = (innerWidth  - BASE_W * scale) / 2;
  const offsetY = (innerHeight - BASE_H * scale) / 2;
  slides.forEach(s => {
    s.style.transform = `translate(${offsetX}px, ${offsetY}px) scale(${scale})`;
  });
}
new ResizeObserver(scaleSlide).observe(document.documentElement);
```

#### 브라우저 호환성

| 기능 | Chrome | Firefox | Safari | iOS Safari |
|------|--------|---------|--------|------------|
| `aspect-ratio` | 88+ | 89+ | 15+ | 15+ |
| `dvh` | 108+ | 109+ | 15.4+ | 15.4+ |
| `min()` / `max()` | 79+ | 75+ | 11.1+ | 11.3+ |
| CSS Grid | 57+ | 52+ | 10.1+ | 10.3+ |
| `inset` | 87+ | 87+ | 14.1+ | 14.5+ |

---

## 3. Agent 2 · 키보드 & 마우스 네비게이션

### 3-1. 키보드 이벤트 처리

#### `keydown` vs `keyup` vs `keypress` 선택 기준

| 이벤트 | 특징 | 권장 여부 |
|--------|------|-----------|
| `keydown` | 누르는 순간 발화, 비문자키(화살표 등) 감지 가능 | ✅ **표준 선택** |
| `keyup` | 떼는 순간 발화, 반복 입력 없음 | ❌ 응답성 낮음 |
| `keypress` | 문자키만 발화, 화살표 감지 불가, Deprecated | ❌ 사용 금지 |

```js
document.addEventListener('keydown', e => {
  // 입력 필드 포커스 시 네비게이션 비활성화
  const tag = document.activeElement?.tagName;
  if (tag === 'INPUT' || tag === 'TEXTAREA' || tag === 'SELECT') return;
  if (document.activeElement?.isContentEditable) return;

  switch (e.key) {
    case 'ArrowRight': case 'ArrowDown': case ' ': case 'PageDown':
      e.preventDefault(); presentation.next(); break;
    case 'ArrowLeft': case 'ArrowUp': case 'PageUp':
      e.preventDefault(); presentation.prev(); break;
    case 'Home': e.preventDefault(); presentation.goTo(0); break;
    case 'End':  e.preventDefault(); presentation.goTo(total - 1); break;
    case 'f': case 'F':
      if (!e.ctrlKey && !e.metaKey) toggleFullscreen(); break;
    case 'Escape': exitFullscreen(); break;
  }
});
```

---

### 3-2. URL Hash 동기화

#### `location.hash` vs History API 비교

| 방식 | 장점 | 단점 |
|------|------|------|
| `location.hash` | 구현 단순, 서버 설정 불필요 | URL에 `#` 포함 |
| History API (`pushState`) | 클린 URL | 서버 fallback 설정 필요 |

```js
// Hash 방식 구현
class HashRouter {
  init() {
    window.addEventListener('hashchange', () => this.#parse(location.hash));
    this.#parse(location.hash);
  }

  update(index) {
    // replaceState: 히스토리에 쌓지 않음
    history.replaceState(null, '', `#slide-${index + 1}`);
    // pushState: 뒤로가기로 이전 슬라이드 복귀 원하면 사용
    // history.pushState({ slideIndex: index }, '', `#slide-${index + 1}`);
  }

  #parse(hash) {
    const m = hash.match(/^#slide-(\d+)$/);
    if (m) {
      const idx = Math.max(0, Math.min(+m[1] - 1, total - 1));
      presentation.goTo(idx, { updateHash: false }); // 무한루프 방지
    }
  }
}
```

---

### 3-3. 터치 / 스와이프 제스처

#### Touch Events vs Pointer Events API 비교

| 방식 | 특징 | 권장 |
|------|------|------|
| Touch Events | 터치 전용, 멀티터치 기본 지원 | 구형 호환 필요 시 |
| Pointer Events API | 터치/마우스/펜 통합, IE11+ | ✅ 신규 프로젝트 |

```js
// Pointer Events 구현 (권장)
let startX = 0, pointerId = null;
const el = document.getElementById('slideViewport');

el.addEventListener('pointerdown', e => {
  if (pointerId !== null) return;
  pointerId = e.pointerId;
  startX = e.clientX;
  el.setPointerCapture(e.pointerId);  // 요소 밖 드래그도 추적
});

el.addEventListener('pointerup', e => {
  if (e.pointerId !== pointerId) return;
  pointerId = null;
  const dx = e.clientX - startX;
  if (Math.abs(dx) > 60) dx < 0 ? presentation.next() : presentation.prev();
});

el.addEventListener('pointercancel', () => { pointerId = null; });
```

```js
// Touch Events 구현 (하위 호환 필요 시)
class TouchSwipeHandler {
  constructor(el, { threshold = 50, velocity = 0.3 } = {}) {
    this.el = el; this.threshold = threshold; this.velocity = velocity;
    el.addEventListener('touchstart', e => {
      this.startX = e.changedTouches[0].clientX;
      this.startY = e.changedTouches[0].clientY;
      this.startTime = Date.now();
    }, { passive: true });  // passive: true → 스크롤 성능 보존

    el.addEventListener('touchend', e => {
      const dx = e.changedTouches[0].clientX - this.startX;
      const dy = e.changedTouches[0].clientY - this.startY;
      const speed = Math.abs(dx) / (Date.now() - this.startTime);
      if (Math.abs(dx) > Math.abs(dy) &&
          (Math.abs(dx) >= this.threshold || speed >= this.velocity)) {
        el.dispatchEvent(new CustomEvent(dx < 0 ? 'swipeleft' : 'swiperight'));
      }
    }, { passive: true });
  }
}
```

---

### 3-4. 진행 표시바 & 슬라이드 번호

```html
<!-- HTML -->
<div class="progress-bar" role="progressbar"
     aria-valuemin="1" aria-valuemax="12" aria-valuenow="3">
  <div class="progress-fill" id="progressFill"></div>
</div>
<div class="slide-counter" aria-live="polite">
  <span id="curNum">3</span> / <span id="totNum">12</span>
</div>
<nav class="dot-nav" role="tablist" aria-label="슬라이드 목록"></nav>
```

```css
/* 진행 표시바 */
.progress-bar  { position: fixed; top: 0; left: 0; width: 100%; height: 3px; background: rgba(0,0,0,.15); z-index: 1000; }
.progress-fill { height: 100%; background: linear-gradient(90deg, #4F46E5, #7C3AED);
                  width: 0%; transition: width 0.35s cubic-bezier(0.4,0,0.2,1); will-change: width; }

/* 도트 */
.dot { width: 8px; height: 8px; border-radius: 50%; background: rgba(255,255,255,.4);
       border: none; cursor: pointer; transition: transform 0.2s, background 0.2s; }
.dot:hover           { transform: scale(1.4); background: rgba(255,255,255,.7); }
.dot[aria-selected="true"] { transform: scale(1.4); background: #fff; }

/* 슬라이드 번호 */
.slide-counter { position: fixed; bottom: 16px; right: 20px;
                  font-size: .875rem; font-variant-numeric: tabular-nums; }
```

```js
function updateUI(index, total) {
  const pct = total <= 1 ? 100 : (index / (total - 1)) * 100;
  requestAnimationFrame(() => {
    document.getElementById('progressFill').style.width = `${pct.toFixed(2)}%`;
  });
  document.getElementById('curNum').textContent = index + 1;
  document.querySelectorAll('.dot').forEach((d, i) => {
    d.setAttribute('aria-selected', i === index ? 'true' : 'false');
    d.setAttribute('tabindex', i === index ? '0' : '-1'); // roving tabindex
  });
}
```

---

## 4. Agent 3 · 슬라이드 전환 애니메이션

### 4-1. CSS Transition vs CSS Animation vs WAAPI 비교

| 항목 | CSS Transition | CSS Animation | Web Animations API |
|------|---------------|---------------|-------------------|
| 사용 난이도 | 쉬움 | 보통 | 보통~어려움 |
| 중간 단계 | ❌ | ✅ | ✅ |
| 일시정지 | ❌ | ✅ | ✅ |
| JS 제어 | 제한적 | 제한적 | 완전 |
| Promise 지원 | ❌ | ❌ | ✅ `.finished` |
| 동적 값 | 어려움 | 어려움 | 용이 |
| **추천 용도** | 단순 hover | 반복/복잡 | 슬라이드 전환 |

---

### 4-2. 대표 전환 효과 구현

#### Fade

```css
.slide { opacity: 0; pointer-events: none; transition: opacity 0.4s ease-in-out; }
.slide.active { opacity: 1; pointer-events: auto; z-index: 1; }
.slide.leaving { opacity: 0; z-index: 0; transition: opacity 0.4s ease-in-out; }
```

#### Slide (translateX)

```css
.slide { transition: transform 0.45s cubic-bezier(0.4,0,0.2,1); }
/* 방향별 클래스 분기 */
[data-direction="next"] .slide[data-state="entering"] { transform: translateX(100%); }
[data-direction="next"] .slide[data-state="entering"].animating { transform: translateX(0); }
[data-direction="next"] .slide[data-state="leaving"].animating  { transform: translateX(-100%); }
```

#### Zoom

```css
@keyframes zoomIn  { from { transform: scale(0.8); opacity: 0; } to { transform: scale(1); opacity: 1; } }
@keyframes zoomOut { from { transform: scale(1);   opacity: 1; } to { transform: scale(1.2); opacity: 0; } }
.slide.zoom-enter { animation: zoomIn  0.4s cubic-bezier(0.34,1.56,0.64,1) forwards; }
.slide.zoom-exit  { animation: zoomOut 0.4s ease-in forwards; }
```

---

### 4-3. WAAPI — 동시 입/퇴장 전환 (권장)

```js
async function transition(fromSlide, toSlide, direction) {
  const enterX = direction === 'next' ? '100%' : '-100%';
  const exitX  = direction === 'next' ? '-100%' : '100%';
  const opts = { duration: 450, easing: 'cubic-bezier(0.4,0,0.2,1)', fill: 'forwards' };

  // GPU 준비
  [fromSlide, toSlide].forEach(s => s.style.willChange = 'transform, opacity');

  // 두 애니메이션 동시 실행 후 완료 대기
  await Promise.all([
    fromSlide.animate(
      [{ transform: 'translateX(0)', opacity: 1 },
       { transform: `translateX(${exitX})`, opacity: 0.5 }], opts
    ).finished,
    toSlide.animate(
      [{ transform: `translateX(${enterX})`, opacity: 0.5 },
       { transform: 'translateX(0)', opacity: 1 }], opts
    ).finished,
  ]);

  // fill:forwards 고스트 스타일 제거
  [fromSlide, toSlide].forEach(s => {
    s.getAnimations().forEach(a => a.cancel());
    s.style.willChange = 'auto';  // GPU 레이어 해제
  });
}
```

---

### 4-4. GPU 가속 활용법

```css
/* ❌ 금지: layout 유발 → 가장 비쌈 */
.slide-bad { transition: left 0.5s; left: 100%; }

/* ✅ 권장: compositor만 사용 */
.slide-good { transition: transform 0.5s; transform: translateX(100%); }

/* will-change: 전환 대상 슬라이드에만 적용 */
.slide[data-state="current"],
.slide[data-state="entering"],
.slide[data-state="leaving"] { will-change: transform, opacity; }

/* 숨겨진 슬라이드는 GPU 레이어 해제 */
.slide[data-state="hidden"] { will-change: auto; }
```

**`will-change` 사용 원칙**
1. 항상 적용하지 말 것 — 메모리 과소비
2. 애니메이션 직전 추가, 완료 후 `auto`로 해제
3. 실제로 변경될 속성(`transform`, `opacity`)만 명시

---

### 4-5. 애니메이션 도중 입력 차단

```js
class PresentationController {
  #isAnimating = false;

  next() {
    if (this.#isAnimating) return;   // 잠금 중이면 즉시 반환
    this.#navigate(this.#cur + 1);
  }

  async #navigate(targetIdx) {
    this.#isAnimating = true;
    this.container.classList.add('is-animating');   // CSS pointer-events 차단

    // 안전장치: transitionend 미발생 대비 타임아웃
    const DURATION = 500;
    const safeTimeout = setTimeout(() => { this.#isAnimating = false; }, DURATION + 100);

    await this.#transition(/* ... */);

    clearTimeout(safeTimeout);
    this.#isAnimating = false;
    this.container.classList.remove('is-animating');
  }
}
```

```css
/* CSS로 마우스/터치 차단 */
.presentation.is-animating { pointer-events: none; }

/* 키보드는 JS의 isAnimating 플래그로만 차단 가능 */
```

---

### 4-6. 이징(Easing) 추천값

| 용도 | cubic-bezier | 설명 |
|------|-------------|------|
| 표준 이동 | `cubic-bezier(0.4, 0, 0.2, 1)` | Material Design Standard |
| 입장 | `cubic-bezier(0, 0, 0.2, 1)` | Decelerate (빠르게 시작 → 느리게 도착) |
| 퇴장 | `cubic-bezier(0.4, 0, 1, 1)` | Accelerate (느리게 출발 → 빠르게 떠남) |
| 스프링 | `cubic-bezier(0.34, 1.56, 0.64, 1)` | 살짝 overshoot하는 bounce 느낌 |

---

## 5. Agent 4 · 슬라이드 내 콘텐츠 애니메이션 (빌드 효과)

### 5-1. Fragment 패턴 — 단계별 등장

#### HTML 구조

```html
<div class="slide">
  <h2>주요 포인트</h2>
  <p class="fragment">첫 번째 항목</p>
  <p class="fragment fade-up">두 번째 항목 (아래에서 올라옴)</p>
  <p class="fragment highlight">세 번째 항목 (강조 색상)</p>

  <!-- data-fragment-index로 순서 명시 (같은 인덱스 = 동시 등장) -->
  <p class="fragment" data-fragment-index="2">A</p>
  <p class="fragment" data-fragment-index="2">B (A와 동시)</p>
</div>
```

#### CSS

```css
/* 기본 숨김 상태 */
.fragment {
  opacity: 0;
  visibility: hidden;
  transform: translateY(18px) scale(0.97);
  transition: opacity 0.4s ease, transform 0.4s ease, visibility 0s 0.4s;
  pointer-events: none;
}

/* 등장 상태 */
.fragment.visible {
  opacity: 1;
  visibility: visible;
  transform: none;
  transition: opacity 0.4s ease, transform 0.4s ease, visibility 0s;
  pointer-events: auto;
}

/* 변형 타입 */
.fragment.fade-up    { transform: translateY(30px); }
.fragment.fade-left  { transform: translateX(30px); }
.fragment.zoom-in    { transform: scale(0.6); }

/* highlight 타입: 처음부터 보이고 색상만 변경 */
.fragment.highlight          { opacity: 1; visibility: visible; transform: none; }
.fragment.highlight.visible  { color: #a78bfa; font-weight: 700; }
```

#### JavaScript — Fragment 관리

```js
class FragmentManager {
  constructor(slide) {
    this.slide = slide;
    this.groups = this.#collectGroups();
    this.currentIndex = -1;
  }

  #collectGroups() {
    const els = [...this.slide.querySelectorAll('.fragment')];
    const map = new Map();
    els.forEach((el, i) => {
      const idx = el.dataset.fragmentIndex !== undefined ? +el.dataset.fragmentIndex : i;
      if (!map.has(idx)) map.set(idx, []);
      map.get(idx).push(el);
    });
    return [...map.entries()].sort(([a], [b]) => a - b).map(([, els]) => els);
  }

  get totalSteps() { return this.groups.length; }
  hasNext() { return this.currentIndex < this.totalSteps - 1; }
  hasPrev() { return this.currentIndex >= 0; }

  next() {
    if (!this.hasNext()) return false;
    this.currentIndex++;
    this.groups[this.currentIndex].forEach(el => el.classList.add('visible'));
    return true;
  }

  prev() {
    if (!this.hasPrev()) return false;
    this.groups[this.currentIndex].forEach(el => el.classList.remove('visible'));
    this.currentIndex--;
    return true;
  }

  showAll() {
    this.groups.forEach(g => g.forEach(el => el.classList.add('visible')));
    this.currentIndex = this.totalSteps - 1;
  }

  hideAll() {
    this.groups.forEach(g => g.forEach(el => el.classList.remove('visible')));
    this.currentIndex = -1;
  }
}
```

#### 슬라이드 전환 시 Fragment 처리 원칙

```js
// 다음 슬라이드로 이동 → 진입 슬라이드의 fragment 모두 숨김 (처음부터 시작)
if (direction === 'next') fragmentManagers[targetIdx].hideAll();

// 이전 슬라이드로 이동 → 진입 슬라이드의 fragment 모두 표시 (이미 다 봤으므로)
if (direction === 'prev') fragmentManagers[targetIdx].showAll();
```

---

### 5-2. 자동 순차 등장 (animation-delay 방식)

슬라이드 진입 시 자동으로 차례대로 등장 — 타이틀 슬라이드, 인트로에 적합

```css
.slide .auto-appear { opacity: 0; transform: translateY(20px); animation: none; }
.slide.active .auto-appear { animation: fadeUp 0.5s ease forwards; }

/* CSS 변수로 delay 동적 부여 */
.slide.active .auto-appear { animation: fadeUp 0.5s ease var(--delay, 0s) forwards; }

@keyframes fadeUp { to { opacity: 1; transform: translateY(0); } }
```

```js
// JS로 delay 변수 주입
function applyStaggerDelay(slide, baseDelay = 0.1, step = 0.2) {
  slide.querySelectorAll('.auto-appear').forEach((el, i) => {
    el.style.setProperty('--delay', `${baseDelay + i * step}s`);
  });
}
```

---

### 5-3. 코드 블록 하이라이트

```html
<!-- 단계별 하이라이트 정의: "줄번호,줄번호 | 다음단계" -->
<div class="code-block" data-highlight-steps="1 | 2,3 | 4,5">
  <pre><code>
    <span class="code-line" data-line="1">function fibonacci(n) {</span>
    <span class="code-line" data-line="2">  if (n &lt;= 1) return n;</span>
    <span class="code-line" data-line="3">  return fibonacci(n - 1)</span>
    <span class="code-line" data-line="4">       + fibonacci(n - 2);</span>
    <span class="code-line" data-line="5">}</span>
  </code></pre>
</div>
```

```css
.code-line {
  display: block;
  padding: 2px 8px;
  margin: 0 -8px;
  border-left: 3px solid transparent;
  transition: background 0.3s ease, opacity 0.3s ease, border-color 0.3s ease;
}
.code-line.hl {
  background: rgba(249, 226, 175, 0.15);
  border-left-color: #f9e2af;
  color: #f9e2af;
}
/* 강조 없는 줄은 흐리게 */
.code-block.has-hl .code-line:not(.hl) { opacity: 0.28; }
```

```js
class CodeHighlighter {
  constructor(block) {
    this.block = block;
    this.steps = (block.dataset.highlightSteps || '')
      .split('|').map(s => s.trim().split(',').map(Number));
    this.currentStep = -1;
  }

  highlight(lineNumbers) {
    this.block.querySelectorAll('.code-line').forEach(l => l.classList.remove('hl'));
    lineNumbers.forEach(n => {
      const line = this.block.querySelector(`[data-line="${n}"]`);
      if (line) {
        line.classList.remove('hl');
        void line.offsetWidth;         // 재애니메이션 위해 reflow 강제
        line.classList.add('hl');
      }
    });
    this.block.classList.toggle('has-hl', lineNumbers.length > 0);
  }

  nextStep() {
    if (this.currentStep >= this.steps.length - 1) return false;
    this.currentStep++;
    this.highlight(this.steps[this.currentStep]);
    return true;
  }

  reset() { this.currentStep = -1; this.highlight([]); }
}
```

---

### 5-4. Syntax Highlighting (순수 JS)

```js
const SyntaxHighlighter = (() => {
  const TOKENS = {
    javascript: [
      { pattern: /(\/\/.*$|\/\*[\s\S]*?\*\/)/gm, class: 'sh-comment' },
      { pattern: /("(?:[^"\\]|\\.)*"|'(?:[^'\\]|\\.)*'|`(?:[^`\\]|\\.)*`)/g, class: 'sh-string' },
      { pattern: /\b(const|let|var|function|return|if|else|for|while|class|new|async|await)\b/g, class: 'sh-keyword' },
      { pattern: /\b(\d+\.?\d*)\b/g, class: 'sh-number' },
      { pattern: /\b([a-zA-Z_$][a-zA-Z0-9_$]*)\s*(?=\()/g, class: 'sh-function' },
    ],
  };

  function tokenize(code, lang) {
    const tokens = TOKENS[lang] || [];
    let result = code.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
    const placeholders = [];
    tokens.forEach(({ pattern, class: cls }) => {
      result = result.replace(pattern, match => {
        const idx = placeholders.length;
        placeholders.push(`<span class="${cls}">${match}</span>`);
        return `\x00${idx}\x00`;
      });
    });
    return result.replace(/\x00(\d+)\x00/g, (_, i) => placeholders[+i]);
  }

  return {
    highlight(container = document) {
      container.querySelectorAll('pre code[data-lang]').forEach(el => {
        el.innerHTML = tokenize(el.textContent, el.dataset.lang);
      });
    }
  };
})();
```

```css
/* Dark 테마 색상 */
.sh-comment  { color: #585b70; font-style: italic; }
.sh-string   { color: #a6e3a1; }
.sh-keyword  { color: #cba6f7; font-weight: bold; }
.sh-number   { color: #fab387; }
.sh-function { color: #89b4fa; }
```

---

### 5-5. 화자 노트 (Speaker Notes)

```html
<!-- 슬라이드 내부에 포함, 발표 중 숨김 -->
<section class="slide">
  <h2>슬라이드 제목</h2>
  <aside class="speaker-notes">
    <p>이 슬라이드에서는 약 2분 정도 설명합니다.</p>
    <p>핵심 포인트: 성능 향상의 이유를 강조할 것.</p>
  </aside>
</section>
```

```css
.speaker-notes { display: none; }   /* 발표 중 완전 숨김 */

/* 발표자 모드 토글 시 */
body.presenter-mode .speaker-notes {
  display: block;
  position: fixed; bottom: 0; left: 0; right: 0;
  background: rgba(0,0,0,0.85); color: #fff;
  padding: 1rem 2rem; border-top: 2px solid #4a9eff;
  z-index: 1000;
}

/* 인쇄 시 포함 */
@media print {
  .speaker-notes {
    display: block !important;
    background: #f0f0f0;
    border: 1px solid #ccc;
    padding: 0.8rem;
    margin-top: 1rem;
    font-size: 0.8rem;
  }
}
```

#### 발표자 뷰 (별도 팝업 창)

```js
// BroadcastChannel으로 탭 간 상태 동기화
const channel = new BroadcastChannel('slide-presentation');

function notifySlideChange(slideIndex, notes, title) {
  channel.postMessage({ type: 'SLIDE_CHANGE', slideIndex, notes, title });

  // window.open으로 연 창에도 직접 전송
  if (notesWindow && !notesWindow.closed) {
    notesWindow.postMessage(
      { type: 'SLIDE_CHANGE', slideIndex, notes, title },
      window.location.origin
    );
  }
}

function openPresenterView() {
  notesWindow = window.open('notes.html', 'presenter-notes', 'width=800,height=500');
}
```

---

## 6. 핵심 패턴 요약

| 주제 | Best Practice (3줄 이내) |
|------|--------------------------|
| **레이아웃** | CSS Grid `place-items: center` + absolute slide stack<br>`width: min(100vw, 100vh * 16/9)` letterbox<br>`clamp()` 반응형 폰트, `100dvh`로 iOS 대응 |
| **네비게이션** | `keydown` 이벤트 + 입력 필드 예외처리<br>Pointer Events API + `setPointerCapture` 스와이프<br>`replaceState` hash 동기화 + `hashchange` 뒤로가기 |
| **전환 애니메이션** | WAAPI `Promise.all`로 입/퇴장 병렬 처리<br>`transform + opacity`만 사용 (layout/paint 회피)<br>`will-change` 전환 직전 추가 → 완료 후 `auto` 해제 |
| **빌드 효과** | `.fragment → .visible` 클래스 토글 패턴<br>다음 이동 시 `hideAll()`, 이전 이동 시 `showAll()`<br>`prefers-reduced-motion`으로 접근성 애니메이션 비활성화 |

---

## 7. 주의사항 & 브라우저 호환성

### 흔한 실수

| 항목 | 잘못된 방법 | 올바른 방법 |
|------|-------------|-------------|
| 위치 이동 | `top/left` transition | `transform: translateX()` |
| 요소 숨김 | `display: none` (레이아웃 붕괴) | `visibility: hidden + opacity: 0` |
| will-change | 모든 슬라이드에 상시 적용 | 전환 직전 추가, 완료 후 `auto` 해제 |
| transitionend | 여러 속성 → 중복 발화 | `e.propertyName === 'transform'` 필터링 |
| fill:forwards | 애니메이션 후 스타일 잠김 | `.cancel()` 호출로 반드시 제거 |
| iOS 높이 | `100vh` (주소창 포함 계산 오류) | `100dvh` + JS `--vh` 변수 폴백 |
| 터치 이벤트 | passive 미지정 | `{ passive: true }` 명시 |
| 코드 highlight | innerHTML 직접 파싱 | `textContent` 읽고 HTML escape 후 주입 |
| fragment 역방향 | 이전 슬라이드 fragments 숨김 유지 | `showAll()` 호출로 모두 표시 |
| 접근성 | 애니메이션 강제 | `prefers-reduced-motion` 미디어쿼리 적용 |

### 브라우저 호환성 일람

| 기능 | Chrome | Firefox | Safari | iOS Safari | IE11 |
|------|--------|---------|--------|------------|------|
| `aspect-ratio` | 88+ | 89+ | 15+ | 15+ | ❌ |
| `dvh` 단위 | 108+ | 109+ | 15.4+ | 15.4+ | ❌ |
| `min()` / `max()` | 79+ | 75+ | 11.1+ | 11.3+ | ❌ |
| `will-change` | 36+ | 36+ | 9.1+ | 9.3+ | ❌ |
| Web Animations API | 36+ | 48+ | 13.1+ | 13.4+ | ❌ |
| Pointer Events | 55+ | 59+ | 13+ | 13+ | 부분 |
| BroadcastChannel | 54+ | 38+ | 15.4+ | 15.4+ | ❌ |
| CSS Grid | 57+ | 52+ | 10.1+ | 10.3+ | 부분 |

> **IE11 지원 필요 시**: Absolute Positioning 레이아웃 + CSS Transition + Touch Events 조합 사용

---

*리서치 날짜: 2026-03-24*
*샘플 코드: `vanilla-slides.html` (동일 디렉토리)*

---

## 8. 적용된 수정사항 (vanilla-slides.html)

> 리서치 이후 `vanilla-slides.html`에 실제로 적용된 변경 내역

### 8-1. 레이아웃: Letterbox → 풀뷰포트

**변경 전 (리서치 초안)**
```css
.presentation {
  display: grid;
  place-items: center;
  width: 100vw;
  height: 100dvh;
}
.slide-viewport {
  width:  min(100vw, 100vh * (16/9));
  height: min(100vh, 100vw * (9/16));
  aspect-ratio: 16 / 9;
}
```

**변경 후 (최종 적용)**
```css
.presentation {
  position: fixed;
  inset: 0;          /* 위아래 좌우 여백 없이 화면 꽉 채움 */
  overflow: hidden;
  background: #000;
}
.slide-viewport {
  position: absolute;
  inset: 0;
  overflow: hidden;
}
```

> 이유: 16:9 letterbox 방식은 와이드 모니터나 세로 모드에서 위아래 여백(검정 바)이 생긴다. 풀뷰포트 방식은 어떤 화면 비율에서도 여백 없이 전체를 채운다.

---

### 8-2. 콘텐츠 폭 제한: `.slide-inner` 추가

```css
/* 슬라이드 내 콘텐츠를 680px로 제한 */
.slide-inner {
  width: 100%;
  max-width: 680px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

/* 내부 컴포넌트 clamp() 축소 */
.slide h1   { font-size: clamp(1.5rem,  3.8vw, 2.6rem); }
.slide h2   { font-size: clamp(1.15rem, 2.6vw, 1.9rem); }
.slide p,
.slide li   { font-size: clamp(0.8rem,  1.4vw, 1.05rem); }
.cards      { gap: clamp(0.45rem, 1.2vw, 0.9rem); }
.card       { padding: clamp(0.7rem, 1.6vw, 1.1rem); }
.code-block { font-size: clamp(0.6rem,  1.2vw, 0.82rem); }
```

```html
<!-- 모든 슬라이드: 콘텐츠를 .slide-inner로 래핑 -->
<section class="slide s1">
  <div class="slide-inner">
    <h1>...</h1>
    <p>...</p>
  </div>
</section>
```

---

### 8-3. 한글 폰트 최적화

```html
<!-- Google Fonts: Noto Sans KR (한글) + Inter (영문) -->
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&family=Noto+Sans+KR:wght@400;700&display=swap" rel="stylesheet">
```

```css
/* 폰트 우선순위: 한글 → 영문 → fallback */
html, body {
  font-family: 'Noto Sans KR', 'Inter', sans-serif;
}

/* 한국어 단어 중간 줄바꿈 방지 */
.slide h1, .slide h2, .slide p, .slide li, .card h3, .card p {
  word-break: keep-all;
}

/* 코드 블록은 모노스페이스 유지 */
.code-block {
  font-family: 'JetBrains Mono', 'Fira Code', Consolas, monospace;
}
```

---

### 8-4. 네비게이션: Fragment 스텝 제거 → 슬라이드 단위 이동

**변경 전 (fragment 스텝 방식)**
```js
next() {
  if (this.#busy) return;
  if (this.#stepFragment()) return;   // fragment를 먼저 소비
  if (this.#cur < total - 1) this.#transition(this.#cur + 1, 'next');
}
```

**변경 후 (슬라이드 단위 이동)**
```js
/* 방향키 한 번 = 슬라이드 한 장 — fragment stepping 없음 */
next() {
  if (this.#busy) return;
  if (this.#cur < this.#total - 1) this.#transition(this.#cur + 1, 'next');
}

/* fragment는 슬라이드 진입 시 자동 순차 등장 (90ms 스태거) */
#revealFragments(idx) {
  [...this.#slides[idx].querySelectorAll('.fragment')]
    .forEach((f, i) => setTimeout(() => f.classList.add('visible'), i * 90));
}

/* 슬라이드 이탈 시 fragment 전부 초기화 */
#hideFragments(idx) {
  this.#slides[idx].querySelectorAll('.fragment')
    .forEach(f => f.classList.remove('visible'));
}
```

> fragment는 이제 "인터랙티브 스텝" 역할이 아닌 "진입 시 순차 등장 연출" 역할. 슬라이드 이동은 항상 한 장씩.

---

### 8-5. 슬라이드 수: 4개 → 10개

| # | 슬라이드 제목 | 주요 컴포넌트 |
|---|--------------|--------------|
| 1 | 타이틀 | badge, h1, subtitle fragment |
| 2 | 핵심 기능 | ul + li fragments |
| 3 | 추천 아키텍처 | 3열 cards (레이아웃/전환/콘텐츠폭) |
| 4 | 레이아웃 전략 | ul + li fragments (fixed/z-index/will-change 등) |
| 5 | 네비게이션 패턴 | 2×2 cards (키보드/터치/URL Hash/Dot) |
| 6 | WAAPI 전환 코드 | code-block (Promise.all 코드) |
| 7 | GPU 가속 최적화 | ul + li fragments (transform/opacity/will-change 등) |
| 8 | Fragment 패턴 | 3열 cards (초기상태/등장효과/슬라이드이동) |
| 9 | 접근성 & 반응형 | ul + li fragments (ARIA/clamp/prefers-reduced-motion 등) |
| 10 | 마무리 | badge, h1, subtitle, 3열 cards 요약 |

---

### 변경 요약

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| 레이아웃 | Letterbox (16:9 고정, 여백 발생) | 풀뷰포트 (`fixed + inset: 0`) |
| 콘텐츠 폭 | 슬라이드 전체 폭 사용 | `.slide-inner` `max-width: 680px` |
| 폰트 | 기본 sans-serif | Noto Sans KR + Inter + `word-break: keep-all` |
| Fragment 동작 | 방향키로 단계별 노출 | 슬라이드 진입 시 자동 스태거 등장 |
| 슬라이드 수 | 4개 | 10개 |

*수정 날짜: 2026-03-24*

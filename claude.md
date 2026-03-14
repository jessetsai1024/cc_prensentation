# 課堂教學網頁簡報 — Claude Code 專案指引

## 專案概述

本專案產出「課堂教學用網頁簡報」。最終產物是**單一 HTML 檔案**，拖進瀏覽器即可播放，無需任何建置工具或伺服器。

用途：現場投影教學，觀眾可能坐在教室後排。

---

## 技術堆疊

| 項目 | 選擇 | 原因 |
|------|------|------|
| 框架 | React 18 (CDN UMD) + Babel Standalone | 單一檔案、零建置、React component 架構 |
| 語言 | JSX（`<script type="text/babel">`） | 直接在 HTML 內寫 React component |
| 字型 | Noto Sans TC + JetBrains Mono (Google Fonts CDN) | 繁中支援 + 等寬程式碼字型 |
| 配色 | 深色主題（Slate 系列底 + 亮色 accent） | 投影環境對比度高、不刺眼 |

### CDN 引入（放在 `<head>`）

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.3.1/umd/react.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.3.1/umd/react-dom.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.26.9/babel.min.js"></script>
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;500;700;900&family=JetBrains+Mono:wght@400;700&display=swap" rel="stylesheet">
```

---

## 檔案結構（全部在一個 .html 內）

```
presentation.html
├── <head>
│   ├── CDN scripts (React, Babel)
│   ├── Google Fonts
│   └── <style> — 所有 CSS（含 CSS Variables）
│
├── <body>
│   └── <div id="root">
│
└── <script type="text/babel">
    ├── Slide Components   — 每一頁是獨立 function component
    ├── SLIDES 陣列        — 註冊所有投影片
    └── App Component      — 主控制器（換頁 + 換步 + 鍵盤監聽）
```

---

## 架構核心：換頁與換步完全獨立

這是本專案最重要的設計決策。換頁和換步是兩套獨立的操作，互不影響。

### 鍵盤快捷鍵

| 按鍵 | 功能 |
|------|------|
| `←` ArrowLeft | 上一頁 |
| `→` ArrowRight | 下一頁 |
| `↑` ArrowUp | 上一步（頁內分步動畫後退） |
| `↓` ArrowDown | 下一步（頁內分步動畫前進） |

### 底部導覽列（4 顆按鈕 + 中間資訊）

```
[ ◀◀ 上一頁 (粉色) ] [ ▲ 上一步 (藍色) ] [ 頁碼 + 步驟圓點 ] [ 下一步 ▼ (藍色) ] [ 下一頁 ▶▶ (粉色) ]
```

- **粉色按鈕 (`nav-btn-page`)** = 換頁操作
- **藍色按鈕 (`nav-btn-step`)** = 換步操作
- 按鈕 disabled 時自動變灰

### 行為規則

1. **換頁時** → `currentStep` 重置為 `0`（該頁從頭開始）
2. **換步時** → 永遠不會跳頁，只在當前頁的 step 範圍內移動
3. **無分步動畫的頁面** → 藍色換步按鈕 disabled
4. **INPUT / TEXTAREA 聚焦時** → 鍵盤事件不觸發（避免打字時換頁）

---

## 投影片 Component 規範

### 每頁的 function signature

```jsx
function SlideXxx({ step }) {
  // step: number — 目前在第幾步，由 App 傳入
  // step === 0 表示初始狀態（剛進入此頁）
  return (
    <div className="slide">
      {/* 內容 */}
    </div>
  );
}
```

### 註冊到 SLIDES 陣列

```jsx
const SLIDES = [
  { component: SlideCover,    totalSteps: 0, title: '封面' },
  { component: SlideContent1, totalSteps: 3, title: '某主題' },
  // ...
  { component: SlideEnd,      totalSteps: 0, title: '結尾' },
];
```

- `totalSteps: 0` → 這頁沒有分步動畫
- `totalSteps: N` → 這頁有 N 步動畫（step 從 0 跑到 N）

### 分步動畫的 pattern

根據 step 值控制元素的顯示/隱藏/動畫：

**Pattern A：逐步顯現（最常用）**

```jsx
// 3 個卡片依序出現
{items.map((item, i) => (
  <div className={`card step-fade ${step > i ? 'visible' : 'pending'}`}>
    {item.content}
  </div>
))}
```

**Pattern B：階段式控制（用於圖表）**

```jsx
const showBars    = step >= 1;  // 第 1 步：長條動畫展開
const showValues  = step >= 2;  // 第 2 步：數值出現
const highlightMax = step >= 3; // 第 3 步：最高分高亮
```

**Pattern C：即時互動（用於計算器、輸入）**

```jsx
const [inputA, setInputA] = useState(5);
const result = someFormula(inputA);
// 互動元素用 step 控制何時出現，但互動本身是即時的
<div className={`step-fade ${step >= 1 ? 'visible' : 'pending'}`}>
  <input value={inputA} onChange={e => setInputA(Number(e.target.value))} />
  <div className="calc-result">{result}</div>
</div>
```

**Pattern D：流程圖節點逐一展開**

```jsx
const nodes = [
  { text: '步驟一', bg: 'var(--accent)' },
  { text: '步驟二', bg: 'var(--accent2)' },
];
{nodes.map((node, i) => (
  <>
    <div className={`flow-box ${step > i ? 'visible' : 'pending'}`}
         style={{ background: node.bg }}>
      {node.text}
    </div>
    {i < nodes.length - 1 && (
      <span className={`flow-arrow ${step > i ? 'visible' : 'pending'}`}>→</span>
    )}
  </>
))}
```

---

## CSS 規範

### CSS Variables（改這裡換整套配色）

```css
:root {
  --bg-primary: #0f172a;        /* 最底層背景 */
  --bg-slide: #1e293b;          /* 投影片背景 */
  --bg-card: #334155;           /* 卡片背景 */
  --text-primary: #f1f5f9;      /* 主文字 */
  --text-secondary: #94a3b8;    /* 副文字 */
  --text-dim: #64748b;          /* 淡色文字、disabled */
  --accent: #38bdf8;            /* 主色（藍）— 換步按鈕 */
  --accent-glow: rgba(56, 189, 248, 0.25);
  --accent2: #f472b6;           /* 副色（粉）— 換頁按鈕 */
  --accent3: #34d399;           /* 綠 — 正確/完成 */
  --accent4: #fbbf24;           /* 黃 — 警告/重點 */
  --danger: #f87171;            /* 紅 — 錯誤 */
  --font-body: 'Noto Sans TC', sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
  --radius: 16px;
  --transition-slide: 0.5s cubic-bezier(0.22, 1, 0.36, 1);
}
```

### 字級規範（現場簡報用）

| 元素 | 字級 | 用途 |
|------|------|------|
| h1 | 64px（封面 80px） | 頁面大標題 |
| h2 | 48px | 投影片標題 |
| h3 | 36px | 卡片/區塊標題 |
| p, li | 28px | 正文 |
| code | 0.9em (~25px) | 行內程式碼 |
| 按鈕 | 24px, min-height 56px | 所有可點擊按鈕 |

**底線：最小字級不低於 24px，按鈕不低於 48px。**

### 重要 CSS class 列表

**投影片容器：**
- `.slide` — 每頁的內容容器（min-height: 100vh, padding: 60px 80px 120px）
- `.title-slide` — 封面特殊版面（置中、脈衝光暈背景）
- `.end-slide` — 結尾頁（置中）

**分步動畫：**
- `.step-fade.visible` — 已顯示（opacity 1, translateY 0）
- `.step-fade.pending` — 未顯示（opacity 0, translateY 20px）
- `.flow-box.visible / .pending` — 流程圖節點
- `.flow-arrow.visible / .pending` — 流程圖箭頭

**卡片：**
- `.card` — 標準卡片
- `.card-accent` — 左側有藍色邊框
- `.card-grid` — 卡片網格排列

**文字高亮：**
- `.highlight` — 藍色粗體
- `.highlight-pink` — 粉色粗體
- `.highlight-green` — 綠色粗體
- `.highlight-yellow` — 黃色粗體

**進場動畫（一次性）：**
- `.animate-in` — slideInUp 動畫
- `.delay-1` ~ `.delay-4` — 延遲 0.1s ~ 0.4s

**互動元素：**
- `.calc-input` — 數字輸入框
- `.calc-result` — 計算結果顯示框
- `.bar-chart` — 長條圖容器
- `.bar-col / .bar / .bar-label / .bar-value` — 長條圖子元素

---

## App 主控制器的結構

```jsx
function App() {
  const [currentSlide, setCurrentSlide] = useState(0);
  const [currentStep, setCurrentStep] = useState(0);

  const slide = SLIDES[currentSlide];

  // 4 個獨立的導航函數
  const goNextPage = ...  // currentSlide + 1, step 重置為 0
  const goPrevPage = ...  // currentSlide - 1, step 重置為 0
  const goNextStep = ...  // currentStep + 1（不動 slide）
  const goPrevStep = ...  // currentStep - 1（不動 slide）

  // 鍵盤監聽
  // ArrowRight → goNextPage
  // ArrowLeft  → goPrevPage
  // ArrowDown  → goNextStep
  // ArrowUp    → goPrevStep

  // 換頁時自動 scrollTop = 0

  return (
    <>
      <ProgressBar />
      <KeyboardHint />
      <SlidesViewport />
      <NavBar />  {/* 4 顆按鈕 + 頁碼 + 步驟圓點 */}
    </>
  );
}
```

---

## 頁面切換動畫

使用 CSS `transform: translateX()` + `transition` 做水平滑動：

```css
.slide-wrapper.current  { transform: translateX(0);     opacity: 1; }
.slide-wrapper.prev     { transform: translateX(-100%);  opacity: 0; }
.slide-wrapper.next     { transform: translateX(100%);   opacity: 0; }
```

transition 時長：`0.5s cubic-bezier(0.22, 1, 0.36, 1)`

---

## 新增一頁投影片的步驟

1. 寫一個新的 function component，接收 `{ step }` prop
2. 在 SLIDES 陣列中加入 `{ component: SlideXxx, totalSteps: N, title: '標題' }`
3. 完成 — App 會自動把它加入導航

---

## 單頁可捲動

每一頁的 `.slide-wrapper` 設定了 `overflow-y: auto`，當內容超過螢幕高度時會出現捲軸。但建議：核心訊息放在第一個螢幕可見範圍內，補充資料才放在下方。

---

## 注意事項

- **所有 CSS 和 JS 都在同一個 HTML 檔案內**，不要拆出獨立的 `.css` 或 `.js`
- **不使用 npm、webpack、vite 或任何建置工具**
- **不使用 localStorage / sessionStorage**（簡報是無狀態的）
- 字型依賴 Google Fonts CDN，如需離線使用請下載字型檔內嵌
- React hooks 的 import：`const { useState, useEffect, useCallback, useRef } = React;`
- 不使用 `import` / `export`（CDN UMD 模式沒有 module system）

---

## 現有模板中的範例頁面

模板 `presentation.html` 內含 7 頁範例，展示以下功能：

| 頁面 | Component | totalSteps | 展示功能 |
|------|-----------|------------|----------|
| 封面 | SlideCover | 0 | 標題動畫 + 脈衝光暈背景 |
| 分步文字 | SlideStepText | 3 | 卡片逐步淡入 |
| 長條圖 | SlideBarChart | 3 | 長條展開 → 數值 → 高亮 |
| 即時計算 | SlideLiveCalc | 2 | 拉桿/輸入 → 即時結果 |
| 流程圖 | SlideFlowchart | 5 | 節點逐一彈出 + 箭頭 |
| 表格 | SlideTable | 3 | 表格 + 可捲動長頁面 |
| 結尾 | SlideEnd | 0 | Q&A 頁面 |

這些範例可以直接複製修改成你需要的內容。

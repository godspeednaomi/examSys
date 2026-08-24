# 試卷生成系統 — 版本修改紀錄

本文件記錄各次修改的目的、遭遇的問題與解決方式。  
專案路徑：`c:\workspace\examSys\`  
主要檔案：`index.html`、`print.css`

---

## v1.0.0 — 初始版本建立

**日期：** 2026-08-24  
**異動檔案：** `index.html`（新增）、`print.css`（新增）

### 功能概要

依據 `ref/跨學段數學題庫生成Prompt設計指南.md` 所定義的 JSON 格式，  
建立一個純前端 Web 應用程式，無需後端伺服器，直接用瀏覽器開啟即可運作。

### 主要功能

| 功能 | 說明 |
|---|---|
| JSON 輸入 | 支援直接貼上文字或上傳 `.json` 檔案 |
| 拖曳上傳 | 可將 `.json` 檔拖曳至上傳區域 |
| 四種輸出版本 | 試題卷、答案卷、詳解卷、教師版 |
| KaTeX 數學公式 | 自動偵測 `$...$`（行內）與 `$$...$$`（區塊）語法並渲染 |
| 試卷 HTML 排版 | 標題、學生資訊欄、作答說明、各大題、選擇題作答欄、底部提醒方塊 |
| 列印 / PDF 輸出 | 透過瀏覽器列印功能匯出 A4 PDF |
| 範例 JSON | 內建一份完整的國小三年級數學考卷範例 |
| JSON Schema 驗證 | 檢查必填欄位並顯示具體錯誤訊息 |

### 技術選擇說明

- **架構：** 純前端 SPA，不依賴後端或 Node.js
- **數學渲染：** KaTeX（CDN 引入），比 MathJax 輕量且渲染速度更快
- **PDF 輸出：** 瀏覽器原生列印引擎（`window.print()`），搭配 `@media print` CSS；
  此方式對中文字型、複雜版面與數學公式的支援最完整
- **印刷版面：** 獨立的 `print.css`，透過 `<link media="print">` 載入，
  僅在列印時生效，不影響螢幕顯示

### 試卷 JSON 結構對應

```
paper_metadata.paper_title      → 試卷大標題
paper_metadata.grade            → 副標題（年級）
paper_metadata.difficulty       → 副標題（難度）
paper_metadata.duration_minutes → 作答說明（建議時間）
paper_metadata.total_score      → 作答說明（滿分）
paper_metadata.tips[]           → 底部「檢查四問」方塊（①②③④）
sections[].section_title        → 大題標題（一、二、三…）
sections[].score_per_question   → 大題標題括號內配分
sections[].questions[]          → 各大題的題目清單
question.stem                   → 題目文字（支援 KaTeX）
question.choices[]              → (A)(B)(C)(D) 選項（選擇題）
question.correct_answer         → 答案（答案卷 / 詳解卷 / 教師版顯示）
question.solution_steps[]       → 逐步解題說明（詳解卷 / 教師版）
question.verification           → 驗算文字（詳解卷 / 教師版）
question.scoring.rubric[]       → 給分規準（教師版）
question.common_mistakes[]      → 常見錯誤（教師版）
question.teacher_tags[]         → 標籤（教師版）
```

---

## v1.1.0 — 修正選擇題選項編號顯示為 `(undefined)`

**日期：** 2026-08-24  
**異動檔案：** `index.html`

### 問題描述

載入外部 JSON 後，選擇題的選項字母顯示為 `(undefined)`，例如：

```
(undefined) 91   (undefined) 97   (undefined) 111   (undefined) 121
```

### 根本原因

`renderChoices()` 函數直接讀取 `c.option_id` 欄位：

```javascript
// 原本的寫法
`(${esc(c.option_id)})`
```

但 AI 生成的 JSON 不一定使用 `option_id` 作為欄位名稱，可能使用 `id`、`label`、`key` 等，  
或完全省略（此時 `c.option_id` 為 `undefined`）。

### 修正方式

加入多欄位名稱容錯（fallback chain），最後以陣列索引自動推導 A/B/C/D：

```javascript
const OPTION_LETTERS = ['A', 'B', 'C', 'D', 'E', 'F'];

const id = c.option_id ?? c.id ?? c.label ?? c.key ?? OPTION_LETTERS[idx] ?? String(idx + 1);
```

同時一併為選項文字欄位與正確性欄位加入容錯：

```javascript
const text      = c.text ?? c.content ?? c.value ?? c.description ?? '';
const isCorrect = c.is_correct ?? c.correct ?? c.isCorrect ?? false;
```

### 同步修正

- `renderMcAnswerGrid()`：作答欄中的正確答案讀取（`correct_answer`）
  加入 `q.answer ?? q.correctAnswer ?? q.correct` 的 fallback

---

## v1.2.0 — 修正正確答案顯示為 `[object Object]`

**日期：** 2026-08-24  
**異動檔案：** `index.html`

### 問題描述

答案卷模式下，部分題目的正確答案顯示為：

```
答：[object Object]
```

### 根本原因

AI 生成的 JSON 中，`correct_answer` 欄位有時是結構化物件而非純字串，例如：

```json
"correct_answer": {
  "number_of_packages": 42,
  "water_per_package": 4,
  "bread_per_package": 5,
  "fruit_per_package": 7,
  "unit": {
    "number_of_packages": "包",
    "water_per_package": "瓶",
    "bread_per_package": "個",
    "fruit_per_package": "個"
  }
}
```

原本的程式碼直接呼叫 `String(correctAnswer)`，物件轉字串後固定輸出 `[object Object]`。

### 修正方式

新增 `anyToStr(val)` 通用轉換函數，依以下優先順序處理各種值類型：

| 優先級 | 情境 | 輸出範例 |
|---|---|---|
| 1 | 純字串 / 數字 / 布林值 | 直接轉字串 |
| 2 | 陣列 | 各元素遞迴轉換，以「；」連接 |
| 3 | 物件含 `text`/`value`/`answer`/`final_answer` 等純文字欄位 | 取第一個非空欄位 |
| 4 | **物件含 `unit` 伴隨物件** | **「數值＋單位」配對，例如：42包、4瓶、5個、7個** |
| 5 | 物件只有純量值（無 unit）| 全部值以「、」連接 |
| 6 | 巢狀物件 | 遞迴展開，以「；」連接 |

```javascript
function anyToStr(val) {
  if (val === null || val === undefined) return '';
  if (typeof val === 'string') return val;
  if (typeof val === 'number' || typeof val === 'boolean') return String(val);
  if (Array.isArray(val)) return val.map(anyToStr).filter(Boolean).join('；');
  if (typeof val === 'object') {
    // 優先嘗試常見文字欄位
    const TEXT_KEYS = ['text','value','content','answer','label','summary',
                       'description','final_answer','result','display','formatted'];
    for (const k of TEXT_KEYS) {
      if (val[k] !== undefined && val[k] !== null) {
        const s = anyToStr(val[k]);
        if (s) return s;
      }
    }
    // 含 unit 子物件 → 值＋單位配對
    if (val.unit && typeof val.unit === 'object') {
      const parts = Object.entries(val)
        .filter(([k, v]) => k !== 'unit' && v !== null && typeof v !== 'object')
        .map(([k, v]) => `${v}${val.unit[k] ?? ''}`);
      if (parts.length) return parts.join('、');
    }
    // 純量值合併
    const scalars = Object.entries(val)
      .filter(([, v]) => v !== null && typeof v !== 'object')
      .map(([, v]) => String(v));
    if (scalars.length) return scalars.join('、');
    // 遞迴
    return Object.values(val).map(anyToStr).filter(Boolean).join('；');
  }
  return String(val);
}
```

### 同步修正

- `q.correct_answer` 的查找鏈加入 `q.final_answer`：
  ```javascript
  q.correct_answer ?? q.final_answer ?? q.answer ?? q.correctAnswer ?? q.correct
  ```
- `renderTeacherBox()` 中的 `criterion`、`common_mistakes` 也改用 `anyToStr()`
- `stem`（題目文字）也改用 `anyToStr()`，並加入 `q.question ?? q.content ?? q.body` fallback

---

## v1.3.0 — 修正解題步驟說明文字顯示為空白

**日期：** 2026-08-24  
**異動檔案：** `index.html`

### 問題描述

詳解卷 / 教師版中，解題步驟的說明文字完全空白：

```
步驟 1：
步驟 2：
步驟 3：
```

### 根本原因

步驟物件的說明欄位原本只嘗試 `description`、`content`、`text`、`detail`，  
但部分 AI 生成的 JSON 使用 `action`、`explanation`、`process`、`reasoning` 等其他名稱，  
導致所有候選欄位都讀不到值，最終顯示為空字串。

### 修正方式

大幅擴充解題步驟各欄位的容錯清單：

**步驟陣列欄位名稱（題目層級）：**
```javascript
q.solution_steps ?? q.steps ?? q.solutionSteps ??
q.solution ?? q.reasoning_steps ?? q.procedure
```

**步驟說明文字欄位（步驟層級）：**
```javascript
step.description ?? step.content  ?? step.text       ?? step.detail     ??
step.action      ?? step.explanation ?? step.explain  ?? step.process    ??
step.reasoning   ?? step.reason    ?? step.work       ?? step.statement  ??
step.note        ?? step.procedure ?? step.instruction ?? step.narration ??
step.analysis    ?? step.solution  ?? step.comment    ?? step.hint       ??
// 最後備援：收集物件中所有未知的字串欄位（排除數字/運算式欄位）
Object.entries(step)
  .filter(([k, v]) => !EXCLUDED_KEYS.includes(k) && typeof v === 'string')
  .map(([, v]) => v).join('　')
```

**步驟數學式欄位（步驟層級）：**
```javascript
step.expression ?? step.formula ?? step.equation ??
step.math       ?? step.calculation ?? step.compute
```

**特殊情況處理：步驟項目為純字串**

部分 AI 生成的 JSON 可能將步驟設計為字串陣列而非物件陣列：

```json
"solution_steps": ["先求18與24的最小公倍數...", "LCM(18,24) = 72", "..."]
```

現在可直接處理，不再報錯或顯示空白：

```javascript
if (typeof step === 'string') {
  html += `<div class="solution-step">
    <span class="step-label">步驟 ${idx + 1}：</span>
    <div class="step-desc">${renderMath(step)}</div>
  </div>`;
  return;
}
```

---

## 已知 JSON 欄位相容性總覽

下表整理目前系統所支援的所有 JSON 欄位別名，方便後續維護參考。

### 題目層級（question object）

| 功能 | 支援的欄位名稱（依優先順序） |
|---|---|
| 題目文字 | `stem`, `question`, `content`, `body` |
| 正確答案 | `correct_answer`, `final_answer`, `answer`, `correctAnswer`, `correct` |
| 解題步驟陣列 | `solution_steps`, `steps`, `solutionSteps`, `solution`, `reasoning_steps`, `procedure` |
| 驗算文字 | `verification`, `check`, `verify` |

### 選項層級（choice object）

| 功能 | 支援的欄位名稱（依優先順序） |
|---|---|
| 選項識別碼 | `option_id`, `id`, `label`, `key`，最後備援：索引推導 A/B/C/D |
| 選項文字 | `text`, `content`, `value`, `description` |
| 是否正確 | `is_correct`, `correct`, `isCorrect` |

### 步驟層級（step object）

| 功能 | 支援的欄位名稱（依優先順序） |
|---|---|
| 步驟編號 | `step`, `step_number`, `number`, `order`, `index`, `id`，最後備援：1-based 索引 |
| 說明文字 | `description`, `content`, `text`, `detail`, `action`, `explanation`, `explain`, `process`, `reasoning`, `reason`, `work`, `statement`, `note`, `procedure`, `instruction`, `narration`, `analysis`, `solution`, `comment`, `hint`，最後備援：所有未知字串欄位 |
| 數學式 | `expression`, `formula`, `equation`, `math`, `calculation`, `compute` |

### 評分規準層級（rubric item）

| 功能 | 支援的欄位名稱（依優先順序） |
|---|---|
| 評分標準文字 | `criterion`, `criteria`, `description`, `text`, `content` |
| 分數 | `score`, `points`, `point`, `marks`, `value` |

---

## 設計決策備忘

### 為何使用純前端而非後端？

瀏覽器的列印引擎（Chrome / Edge 的 Blink）對中文字型、數學公式（KaTeX）與複雜 CSS 的支援最完整，  
輸出的 PDF 品質與人工排版相近，且不需要安裝任何伺服器套件或 Python 環境。

### 為何不使用 jsPDF / html2pdf.js？

這類 Canvas-based 方案會將 HTML 轉為圖片再放入 PDF，導致文字無法搜尋、複製，  
且對中文字型的支援需要額外載入字型包（增加數 MB 體積）。  
瀏覽器列印原生支援向量文字、頁碼與分頁，更適合考卷此類以文字為主的文件。

### `anyToStr()` 的設計原則

AI 生成的 JSON 結構因模型版本、Prompt 寫法而有所不同，  
欄位名稱無法完全統一。`anyToStr()` 的設計原則是「盡量顯示有意義的內容，永遠不崩潰」：  
即使遇到完全未知的結構，也能回傳可讀字串，而非拋出例外或顯示 `[object Object]`。

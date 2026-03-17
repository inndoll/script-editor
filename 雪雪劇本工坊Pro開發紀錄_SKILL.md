# 雪雪劇本工坊 Pro — 專案開發 Skill
**Single-file HTML Screenplay Editor · 繁體中文劇本編輯器**

---

## 專案定位

這是一個**單一 HTML 檔案**的繁體中文劇本編輯器，部署於 GitHub Pages，無需後端、無需安裝任何環境。所有功能、樣式、邏輯都在同一個 `.html` 檔案內。開發者是雪雪（編劇），沒有工程背景，所有開發由 AI 協助完成。

---

## 架構總覽

```
單一 HTML 檔案
├── <head>
│   ├── Google Fonts（Noto Serif TC、Noto Sans TC、Courier Prime）
│   ├── <style>  ── 全部 CSS（含 @media print）
│   ├── docx.js CDN（Word 匯出）
│   └── mammoth.js CDN（Word 匯入）
├── <body>
│   ├── #shell（整體容器）
│   │   ├── #topbar（工具列）
│   │   ├── #main（三欄）
│   │   │   ├── #left（場景／分場卡／角色／筆記／資料）
│   │   │   ├── #center（劇本編輯區）
│   │   │   └── #right（預覽／統計／歷史／格式說明）
│   │   └── #statusbar
│   ├── #modal-overlay（通用 modal，分場卡／角色／筆記編輯）
│   ├── #ctx-menu（右鍵選單）
│   ├── #import-overlay（智慧匯入 modal）
│   └── #guide-overlay（使用指南 modal）
└── <script>  ── 全部 JS（單一合併區塊）
```

### ⚠️ 關鍵架構規則：單一 script 區塊

**所有 JS 必須在同一個 `<script>` 標籤內。**

曾經發生過把 JS 拆成多個 `<script>` 區塊，導致跨區塊的 `function` 宣告無法互相呼叫，頁面一片白。根本原因：每個 `<script>` 標籤是獨立的執行環境，function hoisting 不跨區塊。

合併方式（若需要重新合併）：
```python
# 用 Python 手動解析 <script> 標籤邊界，合併所有 inline script
def find_script_blocks(html):
    blocks = []
    pos = 0
    while True:
        open_start = html.find('<script>', pos)
        if open_start == -1: break
        close_start = html.find('</script>', open_start + 8)
        blocks.append({'start': open_start, 'end': close_start + 9,
                        'js': html[open_start+8:close_start]})
        pos = close_start + 9
    return blocks
```

---

## 初始化順序（關鍵）

所有初始化必須在 `DOMContentLoaded` 裡執行，不可在頂層直接呼叫：

```javascript
window.addEventListener('DOMContentLoaded', function(){
  // 1. 讀取 localStorage 資料到 DB（同步）
  loadFromStorage();
  // 2. Patch renderAll / scheduleUpdate（此時兩個函數都已定義）
  var _origSU = scheduleUpdate;
  scheduleUpdate = function(){ _origSU(); debouncedSave(); };
  var _origRA = renderAll;
  renderAll = function(){ _origRA(); debouncedSave(); autoSnapshot('自動快照'); };
  // 3. 渲染
  renderAll();
  // 4. 同步 DOM 元素（project selector、title fields）
  restoreDomFromStorage();
  // 5. 其他 UI
  renderHistoryPanel();
  var gb = document.getElementById('guide-body');
  if(gb) gb.innerHTML = GUIDE_SECTIONS[0];
});
```

---

## 資料模型

```javascript
let DB = {
  projects: [{id:'p1', name:'劇本名稱'}],
  activeProject: 'p1',
  scripts: {
    p1: {
      id:'s1', projectId:'p1',
      blocks: [/* ScriptBlock[] */]
    }
  },
  sceneCards: { p1: [/* SceneCard[] */] },
  characters:  { p1: [/* Character[] */] },
  notes:       { p1: [/* Note[] */] }
};
```

### ScriptBlock
```javascript
{
  id: 'b1',               // uid() 生成
  type: 'scene',          // 見下方六種類型
  text: 'INT. 咖啡館 - 夜',
  linkedCardId: 'sc1'     // 或 null
}
```

**六種 type**：`scene` / `action` / `character` / `dialogue` / `parenthetical` / `transition`

### SceneCard
```javascript
{
  id, projectId, num,       // 場次編號（自動維護，刪除時重新編號）
  title, location,
  time,                     // '日' | '夜' | '黃昏' | '清晨'
  interior,                 // 'INT' | 'EXT' | 'INT/EXT'
  characters: [],           // string[]
  purpose, conflict, notes,
  linkedBlockId             // 對應劇本中的 scene block id
}
```

### LocalStorage 鍵值
```javascript
var LS_KEY       = 'xiaoyeEditor_v1';   // 整個 DB
var LS_HIST_KEY  = 'xiaoyeHistory_v1';  // 版本歷史
var LS_TITLE_KEY = 'xiaoyeTitle_v1';    // 劇本標題、副標題
```

---

## 核心函數索引

| 函數 | 用途 | 行號約略 |
|------|------|---------|
| `renderAll()` | 重繪所有面板 | ~1070 |
| `renderLines()` | 重繪編輯區所有行 | ~1160 |
| `renderCards()` | 重繪分場卡列表 | ~1235 |
| `onLineKey(e,i)` | 鍵盤事件（Enter/Backspace/Tab/Undo等） | ~1438 |
| `onLineInput(i,el)` | 文字輸入同步到 DB | ~1320 |
| `setLineTypeAt(i,type)` | 切換某行格式 | ~1599 |
| `parseInterior(text)` | 解析 INT/EXT 前綴 | ~1081 |
| `parseTime(text)` | 解析時間後綴（日/夜等） | ~1124 |
| `setInterior(i,interior)` | 點擊 INT/EXT 按鈕 | ~1095 |
| `setTime(i,time)` | 點擊時間按鈕 | ~1135 |
| `importCard(cardId)` | 分場卡導入劇本 | ~1858 |
| `extractScenesFromScript()` | 從劇本主文建立分場卡 | ~2106 |
| `reorderScriptByCards()` | 依分場卡重排劇本 | ~2177 |
| `doUndo() / doRedo()` | Undo/Redo | ~1364/1383 |
| `snapshotUndo()` | 立即快照到 undo stack | ~1348 |
| `exportDocx()` | 匯出 Word（async，需 CDN） | ~2477 |
| `exportFountain()` | 匯出 Fountain 純文字 | ~2375 |
| `exportJson()` | 匯出完整 JSON 備份 | ~2395 |
| `importJson(input)` | 還原 JSON 備份 | ~2422 |
| `saveToStorage()` | 存入 localStorage | ~2771 |
| `loadFromStorage()` | 從 localStorage 讀取 | ~2787 |
| `saveVersion(label)` | 建立版本快照 | ~2829 |
| `newProject()` | 新增劇本專案 | ~2298 |
| `deleteProject()` | 刪除劇本專案 | ~2335 |
| `syncProjectSelector()` | 更新頂部專案下拉選單 | ~2356 |
| `confirmImport()` | 智慧匯入：確認匯入 | ~3101 |
| `classifyLine(text,prevType)` | 規則式段落類型辨識 | ~2981 |

---

## 已知地雷（重要！）

### 1. 巢狀 Template Literal 會讓瀏覽器白頁
三層巢狀：`` `外層 ${`內層`}` ``

Node.js 接受，但 Safari / 某些 Chrome 版本靜默失敗，整個 script 不執行。

**永遠用字串拼接取代：**
```javascript
// ❌ 危險
container.innerHTML = items.map(item => `
  <div onclick="fn('${item.id}')">
    ${subitems.map(s => `<span>${s}</span>`).join('')}
  </div>`).join('');

// ✅ 安全
var html = '';
items.forEach(function(item){
  var inner = '';
  subitems.forEach(function(s){ inner += '<span>' + s + '</span>'; });
  html += '<div onclick="fn(\'' + item.id + '\')">' + inner + '</div>';
});
container.innerHTML = html;
```

### 2. renderAll patch 必須在 DOMContentLoaded 內
```javascript
// ❌ 頂層執行，此時 renderAll 可能未定義
var _origRA = renderAll;

// ✅ 在 DOMContentLoaded 裡執行
window.addEventListener('DOMContentLoaded', function(){
  var _origRA = renderAll;
  renderAll = function(){ _origRA(); debouncedSave(); };
});
```

### 3. HTML 合併 script 用 Python，不用 regex
HTML 內容含有 `[^]*?` 無法用 Python regex 直接匹配（`re.DOTALL` 下有 edge case）。用字串 `find()` / `indexOf()` 替代。

### 4. 刪除分場卡後必須重新編號
```javascript
var remaining = getCards().filter(function(c){ return c.id !== id; });
remaining.forEach(function(c, i){ c.num = i + 1; });
DB.sceneCards[getProject()] = remaining;
```

### 5. `contenteditable` 的原生 undo 和我們的 renderLines() 衝突
瀏覽器的 Ctrl+Z 會嘗試恢復 DOM，但 `renderLines()` 每次都重新生成整個 DOM，兩者互相干擾。必須在 `onLineKey` 裡攔截 Ctrl+Z/Y，呼叫自建的 `doUndo()/doRedo()`，並在 `document.addEventListener('keydown')` 全域也攔截一次（避免 focus 不在行內時漏掉）。

### 6. Enter 斷行需要取得游標位置
```javascript
var caretPos = el ? getCaretOffset(el) : fullText.length;
var textBefore = fullText.slice(0, caretPos);
var textAfter  = fullText.slice(caretPos);
```
`getCaretOffset(el)` 用 `range.cloneRange()` 方式計算，不能用 `selectionStart`（contenteditable 不支援）。

---

## CSS 重點

### 印刷格式（@media print）
- `@page { margin: 1in 1in 1in 1.5in; }` — 好萊塢左1.5吋邊距
- `@page @bottom-center { content: counter(page); }` — 頁碼
- `#editor-page { counter-reset: scene-num; }` — 場景計數器
- `.sl-content[data-type="scene"]::before { content: counter(scene-num) ". "; }` — 場次編號
- `#editor-page::after` — 浮水印（底部中央，rgba 12% 透明度）

### 場景標題行的內外景 / 時間按鈕
- 由 `data-scene="1"` 屬性控制顯示
- `.sl[data-scene="1"] .scene-int-wrap { display: flex; }`
- 切換格式時由 `setLineTypeAt()` 更新此屬性

---

## Word 匯出（exportDocx）

使用 `window.docx`（UMD 格式，從 CDN 載入），需網路連線。

```javascript
var D = window.docx;
var Footer = D.Footer;
var PageNumber = D.PageNumber;
// 頁腳：頁碼 + 浮水印兩行
var footer = new Footer({ children: [
  new Paragraph({ alignment: AlignmentType.CENTER,
    children: [new TextRun({ children: [PageNumber.CURRENT], size: 20 })] }),
  new Paragraph({ alignment: AlignmentType.CENTER,
    children: [new TextRun({ text: '雪雪劇本工坊 Pro（免費版）· 繁體中文劇本編輯器',
      size: 14, color: 'CCCCCC' })] })
]});
// 套用到 section
sections: [{ properties: pageProps, footers: { default: footer }, children: children }]
```

場景行前自動加場次編號：
```javascript
var sceneNum = 0;
// ...在處理 scene block 時：
sceneNum++;
children.push(new Paragraph({
  children: [
    baseRun(sceneNum + '.  ', { bold: true }),
    baseRun(t.toUpperCase(), { bold: true, underline: { type: UnderlineType.SINGLE } })
  ]
}));
```

---

## 智慧匯入辨識規則

```javascript
var SCENE_RE   = /^(INT|EXT|INT\/EXT|內景|外景|室內|室外)[.\s\/]/i;
var SCENE_BODY = /\b(INT|EXT|內景|外景)\b.{0,30}(日|夜|黃昏|清晨|DAY|NIGHT)/i;
var TRANS_RE   = /^(FADE\s*(IN|OUT|TO)|CUT\s*TO|切至|轉場|淡出|淡入)/i;
var PAREN_RE   = /^[(（].{0,50}[)）]\s*$/;
// 全大寫短行（≤32字，無中文標點）→ character
// 前一行是 character/parenthetical/dialogue → dialogue
// 其餘 → action
```

---

## 驗證方法

### 語法檢查
```bash
node << 'EOF'
const html = require('fs').readFileSync('screenplay.html','utf8');
const start = html.indexOf('<script>') + 8;
const end   = html.lastIndexOf('</script>');
try { new Function(html.slice(start, end)); console.log('✓ OK'); }
catch(e){ console.error('✗', e.message); }
EOF
```

### 執行時模擬（含 DOMContentLoaded）
```javascript
const domCbs = [];
const mockWin = {
  addEventListener: (evt, cb) => { if(evt==='DOMContentLoaded') domCbs.push(cb); }
};
// ... 建立 mock document 後執行 script ...
domCbs.forEach(cb => { try{ cb(); }catch(e){ console.error(e.message); } });
```

### 巢狀 template literal 偵測
```javascript
const nested = js.match(/`[^`]*\$\{[^}]*`/g);
if(nested) console.error('發現巢狀 template literal！', nested.length, '處');
```

---

## 功能清單（目前版本）

- ✅ 三欄式佈局（左側欄可收合）
- ✅ 六種劇本段落格式 + 行首格式標籤按鈕（下拉選單）
- ✅ 工具列格式按鈕高亮（反映游標所在行）
- ✅ Tab / Shift+Tab 循環切換格式
- ✅ Enter 斷行（游標位置分行，像 Word）
- ✅ Backspace 合行（游標在行首接回上一行）
- ✅ Ctrl+Z / Ctrl+Y Undo/Redo（自建 80 步 stack）
- ✅ Ctrl+A 全選（游標不在行內時選取整篇劇本）
- ✅ Ctrl+S 手動存檔
- ✅ 場景標題行：INT/EXT 切換按鈕 + 時間切換按鈕（日/夜/黃昏/清晨）
- ✅ Alt+T 快速切換時間
- ✅ 分場卡系統（建立、編輯、刪除後重新編號、拖曳排序）
- ✅ 分場卡導入劇本（自動生成骨架）
- ✅ 從劇本主文導入分場（掃描 scene block 建卡）
- ✅ 依分場卡順序重排劇本
- ✅ 角色資料卡
- ✅ 創作筆記
- ✅ 參考資料上傳（拖曳，支援圖片/PDF/Word/Excel/PPT/HTML/TXT/MD）
- ✅ 右側即時預覽、統計、版本歷史、Fountain 格式說明
- ✅ LocalStorage 自動儲存（1.5秒 debounce）
- ✅ 版本歷史快照（4秒 debounce，最多50版，可回溯）
- ✅ 多專案管理（新建、重命名、刪除）
- ✅ 範例劇本專案（📖 預設內建，含六種格式示範）
- ✅ 智慧匯入（純文字規則辨識 + Word .docx via mammoth.js）
- ✅ 匯出 Fountain（.fountain，Fountain 1.1 標準）
- ✅ 匯出 Word（.docx，好萊塢格式，含頁碼+場次編號+浮水印）
- ✅ 列印 / PDF（好萊塢頁邊距，含頁碼+場次編號+浮水印）
- ✅ 備份 JSON（含所有專案）/ 還原備份
- ✅ 使用指南（11個章節，內嵌 modal）
- ✅ 浮水印：「雪雪劇本工坊 Pro（免費版）· 繁體中文劇本編輯器」

---

## 開發慣例

1. **修改前先 `view` 看清楚目前程式碼**，不要憑記憶 str_replace
2. **修改後一定跑語法驗證**（見上方驗證方法）
3. **所有 HTML 生成用字串拼接，禁用巢狀 template literal**
4. **新功能若需要 DOM 操作，放進 DOMContentLoaded 或函數內，不放頂層**
5. **每次輸出前 `cp` 到 `/mnt/user-data/outputs/雪雪劇本工坊Pro.html`**
6. **工作檔案路徑**：`/home/claude/screenplay.html`

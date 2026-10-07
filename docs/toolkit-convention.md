# LazyHole Toolkit Convention

> 定義 Tools、Commands、MCP 的邊界、職責與命名規範

---

## 1. 現況總覽

```
src/
├── agent/tools/     # 7 個，共 528 行
│   ├── read_file.js
│   ├── write_file.js
│   ├── shell.js
│   ├── web_fetch.js
│   ├── remember.js
│   ├── read_skill.js
│   └── end_session.js
│
├── commands/        # 4 個，共 517 行
│   ├── music.js       # ⚠️ 應改為 MCP
│   ├── memory.js
│   ├── run.js
│   └── help.js
│
└── mcp/             # (未來) Model-Specific Capabilities
    └── music/        # MiniMax 音樂生成
```

---

## 2. 核心原則：三分法

| 類型 | 觸發方式 | 職責 | 範例 |
|------|----------|------|------|
| **Tool** | Agent Function Calling | 通用操作，**模型無關** | `exec_shell`, `read_file`, `web_fetch` |
| **Command** | Telegram `/cmd` | 解析指令 + 格式化輸出 | `/help`, `/run` |
| **MCP** | LLM 原生調用 | **Model-Specific 能力**，換模型可替換 | MiniMax 音樂生成 |

---

## 3. 三者邊界示意圖

```
┌─────────────────────────────────────────────────────────────────┐
│                         Telegram User                            │
└────────────────────────────┬────────────────────────────────────┘
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
    │  /help      │    │  /music     │    │  "幫我查     │
    │  /run       │    │  /memory    │    │   天氣"     │
    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
           │                  │                  │
           ▼                  ▼                  ▼
    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
    │  Commands   │    │  Commands   │    │  Agent Loop │
    │  (薄封裝)    │    │  → MCP      │    │  (意圖/規劃) │
    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
           │                  │                  │
           ▼                  ▼                  ├──────────────────┐
    ┌─────────────┐    ┌─────────────┐          │                  ▼
    │    Tools    │    │     MCP     │    ┌─────────────┐    ┌─────────────┐
    │ (通用操作)   │    │(Model-Specific)│  │    Tools    │    │     MCP     │
    └─────────────┘    └─────────────┘    │ (通用操作)   │    │(Model-Specific)│
                                         └─────────────┘    └─────────────┘
```

---

## 4. Tool — 通用操作層

### 定義

> **模型無關**的操作能力。任何 LLM 都能調用。

### Tool 的本質：Shell 的安全包裝

```
Shell 能力覆蓋：cat、echo、curl、node...
Tool 職責：安全 + 明確 + 可測試
```

| 維度 | Shell | Tool |
|------|-------|------|
| **安全性** | `cat $var` 有 injection 風險 | 參數 schema + 驗證，隔離危險 |
| **明確性** | LLM 要自己想 `cat` 命令 | schema 直接說「需要 path 參數」 |
| **可靠性** | 每次自己處理 error/超時 | Tool 封裝好，統一行為 |
| **可測試** | 難 mock shell 行為 | Tool 函式可以直接 unit test |
| **可替換底層** | 未來要接 S3/DB 得重寫 | 換底層實作，介面不變 |

**結論**：`read_file` / `write_file` 該留，是 **薄封裝 Tool**，不做其他事。

### 特性

| 特性 | 說明 |
|------|------|
| 觸發方式 | Agent Loop（Function Calling） |
| 依賴 | 無特定 LLM 依賴 |
| 部署 | 固定在 `agent/tools/` |
| 回傳 | 結構化結果 `{ success, data, error }` |

### Tool Description 撰寫原則：觸發語境

> **Tool 不只說「能做什麼」，還要說「何時用」**

LLM 需要知道何時該選薄 Tool 而非厚 Shell。description 應包含：
1. **能力描述**：這個 Tool 能做什麼
2. **觸發語境**：什麼情境下應該選這個 Tool（關鍵字、常見意圖）

```js
// ❌ 不足：只說能幹嘛
description: 'Read content from a local file'

// ✅ 充足：說明觸發語境
description: 'Read content from a local file. Use when user asks to "查看", "讀取", "分析" a file, or when you need to inspect code/config before editing or reasoning.'
```

### 現有 Tools 描述修訂

| Tool | 觸發語境關鍵字 | 為什麼用 Tool 而非 Shell |
|------|--------------|--------------------------|
| `read_file` | 「查看」「讀取」「分析」「檢查」+ 檔案 | schema約束、有行號回傳、適合給 LLM 分析 |
| `write_file` | 「寫入」「建立」「修改」+ 檔案 | 目錄自動建立、避免覆蓋、統一錯誤 |
| `web_fetch` | 「搜尋」「查」「抓取」+ URL/主題 | HTML清理、render mode、統一格式 |
| `exec_shell` | **其他所有場景** | 直接跑命令，組合彈性大 |

### Tool 該有的結構

```js
// agent/tools/read_file.js
module.exports = {
  name: 'read_file',
  schema: {
    name: 'read_file',
    description: 'Read content from a local file. Use when user asks to "查看", "讀取", "分析" a file, or when you need to inspect code/config before editing or reasoning.',
    parameters: {
      type: 'object',
      properties: {
        path: { type: 'string', description: 'Absolute path to the file' },
        offset: { type: 'integer', description: 'Line number to start reading (1-based)' },
        limit: { type: 'integer', description: 'Number of lines to read' }
      },
      required: ['path']
    }
  },
  
  async execute({ path, offset, limit }) {
    // 1. 簡單驗證
    if (!path) return { success: false, error: 'path is required' };
    
    // 2. 執行（底層用 Shell）
    const result = await exec_shell(`cat ${path}`);
    
    // 3. 統一回傳格式
    return { success: true, data: result.stdout };
  }
};
```

### ❌ Tool 不該做的

- ❌ **Model-Specific 的能力**（交給 MCP）
- ❌ **Telegram 格式化輸出**（交給 Command）
- ❌ **複雜業務邏輯**（交給 MCP/Command）
- ❌ **直接串接封閉 API**（視情況可封裝，但要有 fallback）

---

## 5. Command — Telegram 介面層

### 定義

> Telegram 指令的解析與格式化包裝。

### 特性

| 特性 | 說明 |
|------|------|
| 觸發方式 | Telegram `/cmd` 前綴 |
| 職責 | 解析參數 → 呼叫 Tool/MCP → 格式化回應 |
| 部署 | `commands/` |
| 回傳 | готов字串，直接給 Telegram |

### ✅ Command 該做的

```js
// commands/music.js
async function handleMusic(args) {
  // 1. 解析參數
  const query = args.join(' ');
  if (!query) return '❌ 請提供關鍵字：/music <關鍵字>';
  
  // 2. 呼叫 MCP（不是 Tool！）
  const result = await mcp.minimax_music.generate({ query });
  
  // 3. 格式化輸出
  return formatMusicResponse(result);
}
```

### ❌ Command 不該做的

- ❌ **直接串接外部 API**（交給 MCP/Tool）
- ❌ **複雜業務邏輯**（交給 MCP/Tool）
- ❌ **Model-Specific 的封裝**（交給 MCP）

---

## 6. MCP — Model-Specific Capability

### 定義

> **特定 LLM 才有**的能力，封裝成可替換的單元。

### 為什麼需要 MCP

```
問題：
如果 Tool 封裝了 MiniMax 音樂生成，
那麼：
- 換成 GPT-4 → Tool 廢了
- 換成 Claude → Tool 廢了
- 只有 MiniMax 能用

解決方案：
把 Model-Specific 的能力抽成 MCP，
每個 LLM 有自己的 MCP 實作，
切換 LLM 時替換 MCP 即可。
```

### 特性

| 特性 | 說明 |
|------|------|
| 觸發方式 | LLM 原生調用（Function Calling + Provider 識別） |
| 依賴 | **強依賴特定 LLM** |
| 部署 | `mcp/{provider}/{capability}.js` |
| 回傳 | 結構化結果 `{ success, data, error }` |

### 建議目錄結構

```
src/
└── mcp/
    └── minimax/
        ├── music/
        │   ├── index.js      # 統一出口
        │   ├── generate.js   # 音樂生成
        │   └── schema.js     # Tool Schema 定義
        ├── tts/
        │   ├── index.js
        │   └── synthesize.js
        └── ...
```

### MCP 介面範例

```js
// mcp/minimax/music/index.js
const provider = 'minimax';
const capability = 'music_generation';

module.exports = {
  name: `${provider}_${capability}`,  // "minimax_music_generation"
  provider,
  capability,
  
  // 工具 Schema（供 LLM 理解如何使用）
  schema: {
    name: 'minimax_music_generation',
    description: 'Generate music based on text description (MiniMax specific)',
    parameters: {
      type: 'object',
      properties: {
        prompt: { type: 'string', description: 'Music description' },
        duration: { type: 'number', description: 'Duration in seconds' }
      },
      required: ['prompt']
    }
  },
  
  // 執行
  async execute({ prompt, duration = 30 }) {
    return await minimaxAPI.generate({ prompt, duration });
  }
};
```

---

## 7. 邊界決策樹

```
收到一個新需求，問自己：

┌─────────────────────────────────────┐
│ Q1: 所有 LLM 都能用嗎？               │
└─────────────────┬───────────────────┘
                  │
      ┌───────────┴───────────┐
      ▼                       ▼
     是                       否
      │                       │
      ▼                       ▼
  ┌─────────┐           ┌─────────┐
  │  Tool   │           │   MCP   │
  │通用操作  │           │Model-Specific│
  └─────────┘           └─────────┘
      │
      ▼
┌─────────────────────────────────────┐
│ Q2: 是 Telegram 指令觸發嗎？         │
└─────────────────┬───────────────────┘
                  │
      ┌───────────┴───────────┐
      ▼                       ▼
     是                       否
      │                       │
      ▼                       ▼
  ┌─────────┐           走 Agent Loop
  │Command  │
  │包裝Tool │
  └─────────┘
```

---

## 8. Tool vs Shell 選用決策表

| 場景 | 選用 Tool | 選用 Shell |
|------|----------|------------|
| 「查看這個檔案的內容」 | ✅ `read_file` | |
| 「建立一個新檔案」 | ✅ `write_file` | |
| 「抓取這個網頁」 | ✅ `web_fetch` | |
| 「列出目錄內容」 | | ✅ `exec_shell` (`ls`) |
| 「建立資料夾」 | | ✅ `exec_shell` (`mkdir`) |
| 「執行 Node.js 腳本」 | | ✅ `exec_shell` (`node`) |
| 「搜尋程式碼」 | | ✅ `exec_shell` (`grep`) |
| 「安裝套件」 | | ✅ `exec_shell` (`npm install`) |
| 「批次處理多個檔案」 | | ✅ `exec_shell` (pipe/combine) |

**快速記憶**：
- **有 schema 約束、常用於推理分析** → Tool
- **組合性強、一次性操作** → Shell

---

## 9. 現有問題與修正

### 🔴 `commands/music.js` (161 行) → 改為 MCP

**現況**：同時包含 MiniMax API 邏輯、格式化

**問題**：與 MiniMax 耦合，換 LLM 就廢

**修正**：
```
commands/music.js    → 薄封裝，呼叫 MCP
mcp/minimax/music/   → MiniMax 專屬實作
```

### 🟡 `commands/memory.js` (244 行)

**現況**：搜尋、格式化、狀態邏輯混在一起

**修正**：
```
commands/memory.js    → 薄封裝
tools/memory_search.js → 通用搜尋（Vector search 實作）
```

### 🟢 `tools/web_fetch.js` (166 行)

**現況**：職責偏多

**修正**：可維持現況，或拆出 `html_parser.js`

---

## 10. 改造優先序

| 優先度 | 檔案/模組 | 動作 | 原因 |
|--------|-----------|------|------|
| 🔴 高 | `commands/music.js` | 重構為 Command + MCP | Model-Specific，不該綁 Tool |
| 🔴 高 | 新增 `mcp/minimax/music/` | 建立 MiniMax 音樂 MCP | 統一 Model-Specific 能力封裝 |
| 🟡 中 | `commands/memory.js` | Command 薄化 | 拆出 Tool |
| 🟢 低 | `tools/web_fetch.js` | 評估是否拆解 | 職責偏多但可用 |
| 🟢 低 | Tool description 更新 | 加入觸發語境 | 讓 LLM 知道何時選 Tool |

---

## 11. 未來擴展：Registry

當工具數量 > 10 之後，可考慮引入 Registry：

```js
// src/toolkit/registry.js
const registry = {
  tools: {
    read_file: require('../agent/tools/read_file'),
    shell: require('../agent/tools/shell'),
  },
  commands: {
    help: require('../commands/help'),
    run: require('../commands/run'),
  },
  mcp: {
    minimax_music: require('../mcp/minimax/music'),
    minimax_tts: require('../mcp/minimax/tts'),
  }
};

// 根據 trigger 自動路由
function route(intent, context) {
  if (isCommand(intent)) return registry.commands[intent](...);
  if (isMCPCapability(intent, context.provider)) return registry.mcp[intent](...);
  return registry.tools[intent](...);
}
```

---

## 12. 快速參考

### ✅ 正確範例

```js
// Tool: 模型無關 + Shell 安全包裝 + 觸發語境
agent/tools/read_file.js
module.exports = {
  name: 'read_file',
  schema: {
    name: 'read_file',
    description: 'Read content from a local file. Use when user asks to "查看", "讀取", "分析" a file, or when you need to inspect code/config before editing or reasoning.',
    parameters: {
      type: 'object',
      properties: {
        path: { type: 'string', description: 'Absolute path to the file' }
      },
      required: ['path']
    }
  },
  async execute({ path }) {
    // 驗證 + 執行 + 統一回傳
    const result = await exec_shell(`cat ${path}`);
    return { success: true, data: result.stdout };
  }
};

// MCP: Model-Specific
mcp/minimax/music/index.js
module.exports = {
  name: 'minimax_music_generation',
  provider: 'minimax',
  async execute({ prompt }) {
    return await minimaxAPI.generate({ prompt });
  }
};

// Command: 薄封裝
commands/music.js
async function handleMusic(args) {
  const result = await mcp.minimax_music.execute({ prompt: args.join(' ') });
  return formatMusicResponse(result);
}
```

### ❌ 錯誤範例

```js
// ❌ 把 Model-Specific 能力做成 Tool
agent/tools/music_generation.js  // 壞味道！綁死 MiniMax

// ❌ Command 直接串 API
commands/music.js
async function handleMusic(args) {
  const result = await minimaxAPI.generate(...); // 應該透過 MCP
}

// ❌ Tool 做格式化
agent/tools/shell.js
return `📦 執行結果：\n${stdout}`; // 格式化應該在 Command

// ❌ Tool description 不寫觸發語境
agent/tools/read_file.js
description: 'Read content from a local file'  // LLM 不知道何時用
```

---

## 13. Tool vs Shell 對照表

| Shell 命令 | Tool | Tool 價值 |
|------------|------|-----------|
| `cat file` | `read_file` | schema 明確、安全驗證、統一錯誤處理 |
| `echo "x" > file` | `write_file` | schema 明確、目錄自動建立、統一錯誤處理 |
| `curl url` | `web_fetch` | HTML 清理、render mode、統一錯誤處理 |
| `ls`, `mkdir`, `node`... | - | 這些直接用 `exec_shell` 即可 |

**結論**：常用的、需要 schema 約束的 → Tool；其他的 → 直接用 Shell。

---

> **最後更新**：2024-01  
> **維護者**：@nigel  
> **版本**：v0.4（新增 Tool Description 觸發語境原則 + Tool vs Shell 選用決策表）

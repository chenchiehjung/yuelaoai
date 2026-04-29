# 月老 yuelaoai 網站 — 工作日誌（交接用）

## 一、專案基本資訊

| 項目 | 內容 |
|---|---|
| **客戶網站** | <https://chenchiehjung.github.io/yuelaoai/> |
| **隱藏管理後台** | <https://chenchiehjung.github.io/yuelaoai/yl-admin-7k3x/> |
| **GitHub repo** | <https://github.com/chenchiehjung/yuelaoai> |
| **GitHub PAT**（fine-grained，僅 Contents 讀寫此 repo） | `[REDACTED-請從原視窗複製]` |
| **管理員密碼** | `zxcvb123`（SHA-256 hash 已嵌入 admin 頁） |
| **使用者** | jamiechen0217@gmail.com |

---

## 二、網站功能（已完成、已上線）

### 客戶端
- 月老問事擲筊頁（單一 HTML 檔，2MB+，含內嵌圖片）
- 表單：姓名 + 國曆生日 + 心中所求
- 擲筊機制：搖手機（DeviceMotion）或點按鈕，模擬筊杯掉落聲音與震動
- 三聖筊累積判定：連續 3 個聖筊才達成
- 失敗流程分階段：
  - 第 1 輪失敗 → 月老未允彈窗（含問題回音 + 歷史摘要）
  - 第 2 輪失敗 → 香爐祈求互動
  - 第 3 輪失敗 → 月老未允彈窗（累積式說明）
  - 第 4 輪失敗 → 拜拜彎腰互動（裝置傾斜偵測，3 次鞠躬，30 度傾斜，每拜震動）
  - 第 5 輪失敗 → 月老未允彈窗（讓使用者拜完後再擲一次才宣告阻礙）
  - 第 6 輪失敗 → 月老阻礙彈窗（流程結束，顯示 LINE CTA）

### 客製化邏輯（最近完成）
- **問題關鍵字偵測**：自動分類為 reunion / marriage / crush / betrayal / family / wealth / newlove / general 等主題
- **金額與時間萃取**：「我這個月能賺到 10 萬業績嗎」會被解析出「10 萬」「這個月」並回顯
- **歷史擲筊紀錄**：累積記錄每輪結果（陰/笑/聖×N），第 2 輪起在彈窗顯示
- **籤詩開場依艱辛度客製化**：0 失敗→「一氣呵成」、4 失敗→「歷香爐淨念」、5+→「經香爐與虔心鞠躬」
- **依陰陽筊比例分流**：偏陰筊 → 「月老保留」風格、偏笑筊 → 「月老試心」風格

### LINE CTA
- 月老阻礙彈窗 + 籤詩頁均有「加入官方 LINE」按鈕（LINE 品牌 SVG 圖標）
- 連到：<https://lin.ee/n2N8qhh>

### 隱私與記憶
- localStorage 自動記住客戶姓名 + 生日（下次自動帶入，問題不存）
- iOS 搖晃還原提示已抑制（送出後把 input 改成 hidden type）

### 隱藏管理員入口
- 主頁面**底部正中央**有一朵 🌸 小花（fixed position、半透明、會擺動）
- **點 3 下**（2.5 秒內）→ 自動跳轉到後台登入頁

### 後台儀表板（admin/index.html）
- 密碼登入（SHA-256 比對，明文 = `zxcvb123`）
- 儀表板顯示：今日訪客、三聖筊成功率、月老阻礙比例、中途離開、LINE 點擊、分享數、平均停留
- 客戶資料表：可搜尋（姓名/問題/籤詩）、篩選結果與日期、排序欄位
- 一鍵匯出 CSV（含 BOM，Excel 開啟中文不亂碼）
- 自動登入（sessionStorage 記住密碼）
- 連線診斷工具（登入頁的「🔧 連線診斷」按鈕）

---

## 三、🚨 ⚠️ 目前進行中的事項（換視窗後請繼續）

### 後端資料庫切換到 Supabase（未完成）

**現況**：
- 目前使用的是 **KVdb.io 匿名儲存**，但用戶測試後**沒有資料進來**，懷疑 KVdb 服務不穩定
- 加了診斷工具想查問題，但用戶已決定**直接換成 Supabase**

**用戶選擇**：Supabase（PostgreSQL，免費，用 GitHub OAuth 登入即可）

### 已給用戶的設定步驟（用戶尚未完成）

1. **STEP 1** — 註冊 Supabase：<https://supabase.com/dashboard/sign-up> → 點 "Continue with GitHub" → 一鍵授權
2. **STEP 2** — 建立 Project：
   - Project name: `yuelaoai`
   - Region: Tokyo 或 Singapore
   - Plan: Free
3. **STEP 3** — 進入 SQL Editor，跑以下 SQL 建表：

```sql
create table customer_records (
  id bigserial primary key,
  timestamp timestamptz default now(),
  name text,
  birthday text,
  question text,
  final_result text,
  final_grade text,
  attempts int default 0,
  duration_seconds int default 0,
  shared boolean default false,
  line_clicked boolean default false,
  history_json text,
  events_json text,
  referrer text,
  user_agent text,
  reason text
);

alter table customer_records enable row level security;

create policy "Anyone can insert" on customer_records
  for insert to anon with check (true);

create or replace function get_yuelao_records(pwd text)
returns setof customer_records
language plpgsql security definer
as $$
begin
  if pwd != 'zxcvb123' then
    return;
  end if;
  return query select * from customer_records order by timestamp desc;
end;
$$;
```

4. **STEP 4** — 從 Project Settings → API 取得：
   - Project URL（如 `https://xxxxx.supabase.co`）
   - anon public key（`eyJhbGciOiJ...` 開頭的長字串）
5. **把這兩個值貼給新視窗的 Claude**

### 新視窗 Claude 接到 ID 後要做的事

1. **修改主網站 `index.html`** — 把現有 KVdb 邏輯換成呼叫 Supabase REST API：
   - `POST {SUPABASE_URL}/rest/v1/customer_records`
   - Headers: `apikey: {ANON_KEY}`, `Authorization: Bearer {ANON_KEY}`, `Content-Type: application/json`
   - Body: 整個 session payload（注意欄位名要從 camelCase 改成 snake_case，例如 `finalResult` → `final_result`、`durationSeconds` → `duration_seconds`、`historyJson` → `history_json` 等）

2. **修改管理頁 `yl-admin-7k3x/index.html`** — 用 RPC 呼叫 `get_yuelao_records`：
   - `POST {SUPABASE_URL}/rest/v1/rpc/get_yuelao_records`
   - Headers 同上
   - Body: `{"pwd": "zxcvb123"}`
   - 回傳的 records 是 snake_case，admin 頁顯示時記得對應

3. **本機 commit + push** 到 GitHub repo（用上面的 PAT）

4. **告訴用戶測試**：強制清快取（`?v=999` 或無痕視窗），跑一輪客戶流程，登入後台驗證資料是否進來

---

## 四、檔案位置（在本機 outputs 資料夾）

| 檔案 | 用途 |
|---|---|
| `outputs/index.html` | 主網站 |
| `outputs/yl-admin-7k3x/index.html` | 隱藏管理頁 |
| `outputs/GitHub-Pages-部署教學.md` | 早期 GitHub Pages 部署教學 |
| `outputs/後台設定指引.md` | 之前 Google Sheets 版本的設定指引（已過時） |
| `outputs/工作日誌.md` | 本檔（交接用） |

---

## 五、開發環境注意事項

- **沙盒網路有限制**：可訪問 github.com，但不能訪問 kvdb.io、supabase.co、api.github.com 等
- **部署方式**：每次改完 `index.html` 後，clone repo → 覆蓋檔案 → commit → push（用 PAT 認證）
- **Edit 工具會 EPERM**：在某些情況下無法直接編輯 outputs 中的檔案，要透過 bash 用 python 寫入

### 標準部署指令（push 用）

```bash
export GH_TOKEN='github_pat_...上面那串...'
cd /tmp && rm -rf yuelaoai
git clone "https://x-access-token:${GH_TOKEN}@github.com/chenchiehjung/yuelaoai.git"
cd /tmp/yuelaoai
cp /sessions/.../mnt/outputs/index.html ./index.html
cp /sessions/.../mnt/outputs/yl-admin-7k3x/index.html ./yl-admin-7k3x/index.html
git config user.email "jamiechen0217@gmail.com"
git config user.name "chenchiehjung"
git add .
git commit -m "..."
git push origin main
```

---

## 六、目前最新 Commit

```
4d46632 — Add KVdb connection diagnostics + robust list parser  (HEAD on main)
5637a53 — Clarify cumulative sheng wording, refine poem journey by yin/xiao tone
f579379 — Personalize interpretations and poem by toss history + question keywords
2fa9116 — Move admin flower to fixed position at bottom of page
8312398 — Use cherry blossom emoji for admin flower
e7f4a8e — Replace 7-tap header trigger with small flower (3 taps)
e675c54 — Add hidden admin trigger
d3bd705 — Switch to KVdb.io (anonymous) for backend storage
343ea0d — Remember user name and birthday via localStorage
b9b2e27 — Add analytics tracking (GA4 + Google Sheets) [obsolete]
...
```

---

## 七、用戶溝通風格備註

- 用戶為非工程背景使用者，溝通用**中文**、**步驟具體**、**術語要解釋**
- 用戶偏好「您幫我做」，不喜歡複雜設定
- 已強烈拒絕 Google 系列服務（Google Sheets / GA4 已建好但未啟用）
- 接受 GitHub OAuth 登入（已用過建 repo + PAT）
- 對「資料庫」「後台」「管理員」等基本概念清楚

---

## 八、待辦清單（按優先順序）

1. ⏳ **【高優先】** 等用戶完成 Supabase 設定 → 接收 Project URL + anon key
2. 🔧 切換主網站與管理頁從 KVdb 改用 Supabase
3. 🧪 部署後請用戶實測一輪流程，確認資料正常進後台
4. 🎨 （未來可做）籤詩內容資料庫擴充（目前籤詩數有限）
5. 🎨 （未來可做）後台增加客戶問題類型分布圖表（圓餅圖等）
6. 🔐 （未來可做）密碼從 `zxcvb123` 升級成更強的密碼

---

## 九、給接手 Claude 的話

1. **打招呼後立刻問用戶**：「Supabase 那邊有設定好了嗎？拿到 Project URL 和 anon key 了嗎？」
2. 拿到 IDs 後就照「三、未完成事項 / 新視窗 Claude 接到 ID 後要做的事」執行
3. 用戶情緒：可能有點疲倦（KVdb 沒進資料讓他失望了），請有耐心、不要再叫他做太多步驟、做完馬上驗證能用
4. 用戶可能還會有後續需求（例如客製化內容、UI 調整、加新功能），靈活應對

祝接手順利 🌸

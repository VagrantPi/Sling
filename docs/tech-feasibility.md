# Sling：手機連結收藏 App 技術選型與可行性分析

> 調研日期：2026-11-27
> 範圍：分享選單入口、系統架構（是否需要 server）、內容解析、自動分類、競品與產品可能性
> 標註「⚠️ 未驗證」的項目代表調研時無法取得一手來源，實作前需自行實測或確認。

---

## 0. TL;DR：每個問題的結論

| 問題 | 結論 |
|---|---|
| 分享選單裡的 App 選項怎麼出現？ | Android 只要在 Manifest 宣告 `ACTION_SEND` + `text/plain` 的 intent-filter，App 就會出現在 Sharesheet。頂端「人頭／頻道」那一排是 **Sharing Shortcuts（Direct Share）**，可以**每個分類各發布一個 shortcut**，分享時一鍵就存進該分類。排序由系統依使用頻率決定，使用者可以長按 Pin。iOS 要用 Share Extension，PWA 只在 Android Chrome 能當分享目標。 |
| 需要 server 嗎？ | **個人 MVP 不需要**。手機端就能完成儲存、抓 metadata、規則分類。只有在以下情況才需要極薄的後端：要呼叫雲端 LLM 又不想把 API key 放在 App 裡、要多裝置同步／網頁版、要做 LINE/Telegram Bot 入口、要推播每週回顧。建議 **local-first，後端可選且可延後**。 |
| 分類需要 AI 嗎？ | **不一定**。建議分層：使用者選擇（Direct Share 分類捷徑）→ 規則（網域／關鍵字）→ 裝置端 embedding kNN → 低信心才交給雲端 LLM。個人用量（10–30 則／天）下，雲端小模型每月成本約 $0.2–2 美元，幾乎可以忽略。真正的難點是**社群平台抓不到內容**，不是分類演算法。 |
| 市場上有哪些產品？ | Raindrop.io、Instapaper、Readwise Reader、Matter（無 Android）、Karakeep（開源自架、可接 LLM）、Linkwarden、Wallabag、Cubox 等。Pocket 已於 2025-07-08 關閉，Omnivore 已於 2024 年底關閉，LINE Keep 已於 2024-08-28 結束。 |
| 有產品化可能嗎？ | **當大眾商業產品：風險高，不建議重押**。**當個人工具／開源專案／作品集：非常適合**。唯一值得驗證的商業切入點是「**繁中社群內容（Threads/PTT/Dcard/LINE）可靠擷取 + 主動回顧，以 Bot 形式切入**」，目前仍是假設。 |

---

## 1. 需求拆解

使用者目前的流程（從截圖來看）：

```
App 內瀏覽器（Custom Tab，「在 Brave 中執行」）→ ⋮ → 分享… → Sharesheet
  → 選 LINE / Messenger → 選自己 → 送出 → 回到原 App
```

痛點有三個：
1. **入口太長**：要跳到通訊軟體、找到自己、送出、再切回原 App。
2. **沒有分類**：所有連結混在聊天記錄裡，越存越多就找不到。
3. **存了不看**：這是整個 read-later 品類的結構性問題（見第 6 節）。

目標流程：

```
分享… → Sharesheet 點「Sling · 技術」（或點 Sling 再選分類）→ Toast「已存」→ 留在原 App
                                                    ↓（背景）
                              URL 正規化 / 去重 → 抓 metadata → 自動分類（若未指定）
```

---

## 2. 分享選單入口

### 2.1 Android：成為 Share Target

```xml
<!-- AndroidManifest.xml -->
<activity
    android:name=".share.ShareReceiverActivity"
    android:exported="true"
    android:excludeFromRecents="true"
    android:theme="@style/Theme.Sling.Translucent">
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
    <!-- 選用：讓 Android 10 以下也能用 Direct Share，需要 androidx.sharetarget:sharetarget 依賴 -->
    <meta-data
        android:name="android.service.chooser.chooser_target_service"
        android:value="androidx.sharetarget.ChooserTargetServiceCompat" />
</activity>

<!-- 在 launcher activity 上 -->
<meta-data android:name="android.app.shortcuts" android:resource="@xml/shortcuts" />
```

Intent 帶來的資料：

| Extra | 內容 | 注意 |
|---|---|---|
| `EXTRA_TEXT` | 主要資料 | 很多 App 會塞成「貼文文字 + 空白 + URL」，要用 regex `https?://\S+` 把 URL 拆出來，剩下的文字留作標題／內文訊號 |
| `EXTRA_SUBJECT` / `EXTRA_TITLE` | 標題 | 不一定有，有無看來源 App |

**最佳 UX：半透明、不開主 App 的 Activity**
1. 在 `onCreate` 解析 intent，寫入 Room。
2. 顯示 Toast，或一個小 BottomSheet（可選分類），然後立刻 `finish()`。
3. 抓取 metadata、分類交給 `WorkManager` 在背景執行。
4. **不要** `startActivity(MainActivity)`，使用者才會留在原本的 App。

### 2.2 Direct Share：用「分類捷徑」取代 AI 分類（本報告的關鍵建議）

截圖頂端那一排（Messenger 聯絡人、Slack 的 `a-backend`、`綠果計畫` 頻道）是 **Sharing Shortcuts API**。Android 11 起 `ChooserTargetService` 已棄用，Direct Share 只能用這套 API。

Slack 把「每個頻道」各發布成一個 shortcut。Sling 也可以照做：**每個分類各發布一個 shortcut**，例如「Sling · 技術文章」「Sling · 影片」「Sling · 美食」。這樣分類就在分享的那一刻由使用者一鍵完成：
- 不需要 AI，準確率 100%
- 這也是最好的訓練資料：使用者自己選的分類可以拿來訓練後續的 kNN 自動分類

```xml
<!-- res/xml/shortcuts.xml -->
<shortcuts xmlns:android="http://schemas.android.com/apk/res/android">
    <share-target android:targetClass="app.sling.share.ShareReceiverActivity">
        <data android:mimeType="text/plain" />
        <category android:name="app.sling.category.SAVE_TO_CATEGORY" />
    </share-target>
</shortcuts>
```

```kotlin
fun publishCategoryShortcut(ctx: Context, cat: Category) {
    val sc = ShortcutInfoCompat.Builder(ctx, "cat_${cat.id}")   // 用穩定 ID，不要重複使用
        .setShortLabel(cat.name)
        .setLongLived(true)
        .setCategories(setOf("app.sling.category.SAVE_TO_CATEGORY"))
        .setIntent(Intent(ctx, ShareReceiverActivity::class.java)
            .setAction(Intent.ACTION_SEND)
            .putExtra("category_id", cat.id))
        .setIcon(IconCompat.createWithResource(ctx, cat.iconRes))
        .build()
    ShortcutManagerCompat.pushDynamicShortcut(ctx, sc)          // 會一併回報使用紀錄，有助排名
}
// 刪除分類時：ShortcutManagerCompat.removeLongLivedShortcuts(ctx, listOf("cat_$id"))
```

限制：
- 每個 App 的 dynamic shortcut 有上限（`ShortcutManagerCompat.getMaxShortcutCountPerActivity()`，通常約 4–5 個），所以只發布**最常用的前 N 個分類**，其他分類走「點 Sling → BottomSheet 選分類」。
- 排名完全由系統依使用頻率決定，App 不能強制置頂。⚠️ 未驗證：在 Motorola / Android 14–16 上實際會不會出現在頂端那排，需要實測。
- 使用者可以長按 Sharesheet 裡的 App 選「固定」（Android 11 恢復此功能）。⚠️ 未驗證：各 OEM 新版本的行為。

### 2.3 其他 Android 入口（備選）

| 入口 | 用途 |
|---|---|
| `ACTION_PROCESS_TEXT` | 選取一段網址文字時，出現在浮動選單 |
| Quick Settings Tile | 點一下就儲存剪貼簿內容（Android 10 起背景讀剪貼簿受限，要在前景 Activity 讀取） |
| Telegram / LINE Bot | 零安裝入口，「傳給自己」的習慣不用改（需要 server，見第 3 節） |

### 2.4 iOS：Share Extension

- Xcode 新增 Share Extension target，`NSExtensionActivationRule` 設 `NSExtensionActivationSupportsWebURLWithMaxCount = 1` 和 text。
- 用 **App Group** 與主 App 共享 SQLite 或 UserDefaults。
- Extension 記憶體上限很低（⚠️ 常見說法約 120MB，未驗證），所以 extension 只存 URL 和文字，抓取和分類都交給主 App 或背景任務。
- 長期安裝需要 Apple Developer Program（$99/年）。免費帳號 sideload 7 天就過期。
- 零程式碼替代：iOS 捷徑（Shortcuts）可以接收分享的 URL，再轉送到 API 或 Bot。

### 2.5 PWA：Web Share Target

- Android Chrome：安裝成 PWA 後，用 manifest 的 `share_target` 就能出現在分享選單。
- iOS Safari：**不支援**當分享目標（WebKit bug 194593 從 2019 年開到現在）。
- 結論：PWA 可以當 Android 免上架的快速原型，但做不到 Direct Share，也做不到不開畫面的一鍵儲存，**不建議當主要方案**。

### 2.6 框架選擇

| 選項 | 適合情境 | 說明 |
|---|---|---|
| **原生 Kotlin + Jetpack Compose** ✅ | 主力手機是 Android、追求最佳分享體驗 | 半透明 Activity、Sharing Shortcuts、WorkManager 都是一等公民，一兩天內就能做出可用的版本 |
| Flutter（`receive_sharing_intent`） | 一開始就要雙平台 | 套件仍有維護（v1.9.0）。但它主要處理「分享進來就開主 App」的流程，要做不開畫面的儲存或 Direct Share，仍然得寫原生程式 |
| Expo（`expo-share-intent`） | 熟 RN / TS | 不能用 Expo Go，要 prebuild。限制和 Flutter 類似 |
| Kotlin Multiplatform | 之後要擴 iOS 又想共用邏輯 | 共用解析、分類、資料層，分享入口仍各自寫原生 |

**建議**：MVP 用原生 Kotlin。把解析、分類邏輯寫成純 Kotlin 模組，之後要做 iOS 時可以平滑升級到 KMP。

---

## 3. 需要 server 端嗎？

### 3.1 各功能可以放在哪裡

| 功能 | 手機端可行？ | 什麼情況需要 server |
|---|---|---|
| 接收分享、儲存 | ✅ Room/SQLite | — |
| URL 正規化、去重、追 redirect | ✅ | — |
| 抓 OG / oEmbed / JSON-LD | ✅ Jsoup + OkHttp，**而且手機效果更好**（住宅 IP 不易被擋；WebView 可以帶使用者自己的登入狀態） | 規模化之後，文章類網站可以由 server 補抓全文 |
| 規則分類、embedding kNN | ✅ ONNX Runtime Mobile | — |
| 雲端 LLM 分類 | ⚠️ 技術上可以直接打 API | **API key 不應該放進 APK**（可以被反編譯取出）。純自用時把自己的 key 放在自己手機上是可接受的風險；只要發給其他人，就一定要有 proxy |
| 多裝置同步、網頁版閱讀 | ❌ | 需要後端或 BaaS |
| 每週回顧推播 | 🔶 本地 `WorkManager` 通知就做得到 | 跨裝置推播或寄 email 才需要 |
| LINE / Telegram Bot 入口 | ❌ | 一定要有 webhook server |

### 3.2 架構演進路線

```
Phase 1（自用 MVP）           Phase 2（加 AI、多裝置）              Phase 3（產品化）
┌──────────────┐            ┌──────────────┐                    ┌──────────────┐
│ Android App  │            │ Android App  │──sync──┐           │ Android/iOS  │
│ Room + WM    │            │ local-first  │        ▼           └──────┬───────┘
│ Jsoup 抓取    │            └──────┬───────┘   ┌─────────┐              │
│ 規則 + 捷徑分類 │                   │ LLM      │Supabase │      ┌───────▼────────┐
└──────────────┘                   ▼          │/Postgres│      │ API (Workers / │
  無 server                  ┌──────────────┐  └─────────┘      │  Cloud Run)    │
                            │ CF Worker    │                    │ + 抓取 worker   │
                            │ (LLM proxy)  │                    │ + Bot webhook  │
                            └──────────────┘                    │ + 排程回顧      │
                                                                └────────────────┘
```

**我的架構建議**：
- **Local-first**：手機上的 SQLite 是 source of truth，server 只是同步目標與 AI proxy。這樣離線可用，server 掛掉也不影響儲存。
- Phase 2 的後端選 **Cloudflare Workers（+ D1）** 或 **Supabase**。前者在個人用量下幾乎免費，適合當 LLM proxy 和 Bot webhook；後者附帶 Auth、Postgres 和 pgvector，以後做語意搜尋比較方便。只是為了藏 API key 的話，Workers 就夠了。
- 同步策略：每筆記錄帶 `updated_at` + `deleted_at`（軟刪除），採 last-write-wins 就夠用。單人使用不需要 CRDT。

### 3.3 資料模型草案

```sql
CREATE TABLE category (
  id          TEXT PRIMARY KEY,          -- UUID，用 ID 關聯，改名才不會壞
  name        TEXT NOT NULL,
  description TEXT,                      -- 給 LLM / embedding 種子用
  centroid    BLOB,                      -- embedding 質心（Phase 2）
  sort_order  INTEGER,
  updated_at  INTEGER, deleted_at INTEGER
);

CREATE TABLE link (
  id              TEXT PRIMARY KEY,
  url             TEXT NOT NULL,         -- 原始 URL
  normalized_url  TEXT NOT NULL UNIQUE,  -- 去 utm/fbclid/igsh、統一 youtu.be 等
  shared_text     TEXT,                  -- EXTRA_TEXT 去掉 URL 後的文字（社群貼文的關鍵訊號）
  source_app      TEXT,                  -- 來源 App 的 package（可從 referrer 推測）
  title TEXT, description TEXT, image_url TEXT, site_name TEXT,
  fetch_status    TEXT,                  -- pending / ok / blocked / failed
  category_id     TEXT REFERENCES category(id),  -- NULL = 未分類
  classified_by   TEXT,                  -- user / shortcut / rule / knn / llm
  confidence      REAL,
  status          TEXT DEFAULT 'unread', -- unread / read / archived
  created_at INTEGER, read_at INTEGER, updated_at INTEGER, deleted_at INTEGER
);

CREATE TABLE rule (
  id TEXT PRIMARY KEY,
  kind TEXT,                             -- domain / keyword / source_app
  pattern TEXT,
  category_id TEXT REFERENCES category(id)
);
```

---

## 4. 內容解析

### 4.1 資料來源優先順序

1. **Share intent 自帶的文字**：免費、最可靠。社群貼文通常抓不到 OG，這段文字往往是唯一的好訊號。
2. **OG / Twitter Card / JSON-LD / `<title>`**：用 Jsoup 解析。
3. **oEmbed**：YouTube（`youtube.com/oembed`）和 X（`publish.twitter.com/oembed`）不需要 key。Facebook/IG 的 oEmbed 需要 Meta App Review，個人用不實用。⚠️ Threads 是否有公開 oEmbed，未驗證。
4. **全文擷取**（Readability）：只對文章類網站做（掘金、Medium、PTT、部落格）。

### 4.2 一定會遇到的坑

| 問題 | 對策 |
|---|---|
| IG / FB / Threads / X / 小紅書沒登入只回傳通用 OG 或登入頁 | **接受降級**：用 intent 文字 + 網域判斷分類。進階做法是用 WebView 帶使用者自己的登入狀態渲染頁面（維護成本高，放在 Phase 3） |
| 短網址（t.co、lin.ee、xhslink、fb.me） | 追 redirect，設最大跳轉次數和 timeout |
| 追蹤參數 | 移除 `utm_*`、`fbclid`、`igsh(id)`、`gclid`、`si`、`xsec_token` 等，host 轉小寫、去掉 `www.` 和 `m.`、去掉 fragment |
| 重複收藏 | 對 `normalized_url` 建 UNIQUE。YouTube 的 `watch?v=`、`youtu.be/`、`shorts/` 統一成同一個 video ID |
| 第三方擷取服務（Jina Reader、Microlink、Firecrawl） | 對一般文章有幫助，**對社群平台一樣會被擋**，所以不是解方 |

**架構上的關鍵**：從 server 端抓社群平台（資料中心 IP）幾乎一定會被擋，從手機端抓反而比較好。這也是「不需要 server」的另一個理由。

---

## 5. 分類

### 5.1 方案比較

| 方案 | 準確度 | 成本 | 隱私 / 離線 | 支援使用者自訂分類 | 實作 |
|---|---|---|---|---|---|
| **A. 分享時選（Direct Share 分類捷徑）** | 100% | 0 | ✅ / ✅ | 原生支援 | 低 |
| **B. 規則**（網域、關鍵字、來源 App） | 命中時高 | 0 | ✅ / ✅ | 要手動建規則 | 低 |
| **C. 裝置端 embedding + kNN** | 中→高，資料越多越準 | 0 | ✅ / ✅ | 免重訓，新增分類即可用 | 中 |
| **D. 裝置端 LLM**（Gemini Nano） | 中 | 0 | ✅ / ✅ | prompt 放分類清單 | 中，**裝置覆蓋率低** |
| **E. 雲端 LLM**（Haiku 級） | 高 | 極低 | ❌ / ❌ | 最好，冷啟動也行 | 低 |

### 5.2 建議：分層 pipeline

```
使用者有選分類？ ──是──► 完成（classified_by = user/shortcut）
      │否
      ▼
規則命中？（youtube.com → 影片） ──是──► 完成
      │否
      ▼
kNN 相似度 ≥ 門檻且領先第二名 ≥ margin？ ──是──► 完成（Phase 2）
      │否
      ▼
雲端 LLM（結構化輸出，category_id 用 enum 限定在既有 ID + "none"）
      │
      ▼
信心低 → 未分類，進入「待整理」收件匣，讓使用者一鍵修正
使用者修正 → 更新該分類的範例向量；同一網域修正 3 次 → 建議建立規則
```

### 5.3 雲端 LLM 細節

- 輸入只送 `標題 + 描述 + shared_text + 前 ~1000 字`，不送 cookie 或完整頁面。
- 用 tool use 或 JSON schema 約束輸出：
  ```json
  { "category_id": "<enum: 既有 ID | none>", "confidence": 0.0, "suggested_new_category": "string|null" }
  ```
- 成本估算（每則約 1,500 input + 50 output tokens）：依模型等級，每 1,000 則約 **$0.2–$1.75**。每天 30 則（約 900 則／月）**每月不到 $2**。分類清單放在 prompt 前段，可以用 prompt caching 再省一些。⚠️ 價格以各家官方定價頁為準。中文 token 數偏高，需要實測。
- 裝置端 LLM（Gemini Nano / ML Kit GenAI）只有部分旗艦機支援。⚠️ 一般 Motorola 機型是否支援，未驗證。所以只能當「有就用」的加分項。

### 5.4 Embedding（Phase 2）

- 中文需要多語模型，例如 `multilingual-e5-small`（量化後約百 MB 級），用 ONNX Runtime Mobile 執行。MediaPipe 內建模型以英文為主，中文效果需要實測。
- 新增分類時，用「名稱 + 描述」產生種子向量，立刻就能分類（解決冷啟動）。
- 累積 100–200 筆已分類資料後，再評估 kNN 能取代多少比例的 LLM 呼叫。

### 5.5 分類生命週期

| 操作 | 行為 |
|---|---|
| 建立 | 填名稱（選填一句描述）→ 產生種子向量 → 若進入前 N 常用就發布 Direct Share shortcut → 可選「把未分類的項目重新分類」 |
| 刪除 | 項目**移到未分類**，不連帶刪除。刪除前顯示影響筆數。移除對應的 shortcut |
| 合併 | 項目、規則、範例向量都轉到目標分類 |
| 改名 | 只改 `name`，ID 不變，並更新 shortcut label |
| AI 建議新分類 | 只建議，不自動建立。未分類裡累積了多筆相似項目時，才提示一次 |
| 單選或多標籤 | **主分類單選**（UX 簡單），標籤之後再說 |

---

## 6. 市場與競品

### 6.1 已經倒下的

| 產品 | 狀態 |
|---|---|
| **Pocket**（Mozilla） | 2025-05-22 宣布，**2025-07-08 關閉**，2025-10-08 永久刪除資料。官方理由是「人們使用網路的方式改變」，要把資源集中在 Firefox |
| **Omnivore**（開源） | 2024 年底團隊加入 ElevenLabs，服務關閉 |
| **LINE Keep** | **2024-08-28 結束**，「Keep 筆記」繼續提供（取消 1GB 免費空間） |

### 6.2 現存主要競品

| 產品 | 平台 | 價格 | AI 分類 | 開源 | 備註 |
|---|---|---|---|---|---|
| **Raindrop.io** | 全平台 | 免費 / Pro 約 $28/年 | Pro 有建議標籤 | 否 | Pocket 關閉後最大受益者，**最接近這次的需求，可以馬上用** |
| Instapaper | 全平台 | 免費 / 付費 | 弱 | 否 | 老牌，以閱讀為主 |
| Readwise Reader | 全平台 | 約 $10–13/月 | 有（摘要、標註） | 否 | 付費模式成功的代表，目標是重度閱讀者 |
| Matter | iOS / web | 免費 / Premium | 有 | 否 | **沒有 Android** |
| **Karakeep**（前 Hoarder） | 全平台 + 自架 | 免費（AGPL） | ✅ 可接 LLM 自動標籤、摘要 | ✅ | 技術人首選，社群公認是 Omnivore 的繼任者 |
| Linkwarden | web / PWA | 自架免費 / 雲端約 $3/月 | 有 | ✅ | |
| Wallabag | 全平台 | 自架免費 / 託管版 | 無 | ✅ | |
| Cubox | 全平台 | 免費 / Pro 約 $29–49/年 | 有 | 否 | 中文市場、AI |
| mymind / Fabric / Recall / Glasp | 多為 web + iOS | 付費為主 | AI 主打 | 否 | ⚠️ 2026 現況未完整驗證 |
| 2025–26 新進者 | 多為 iOS 優先 | — | AI 標籤、摘要 | 否 | SmartMark、Linkwise 等一大批小型 App，**沒有看到針對 Threads 或繁中社群的產品**（⚠️ 搜尋範圍有限） |

### 6.3 市場訊號

**難做的原因**
- Pocket（大公司砍掉）、Omnivore（被收購後關閉）兩個前車之鑑，說明這類產品很難獨立變現。
- 「存了不看」是結構性問題：儲存很便宜，閱讀很貴。
- 免費替代品很多：Chrome/Safari 閱讀清單、Telegram 已儲存訊息、LINE Keep 筆記、Notion。
- 付費意願低，價位大約落在每年 $28–120。

**能活下來的模式**：有明確付費族群（Readwise：重度閱讀與標註）、freemium 加上極低成本（Raindrop）、高價小眾（mymind）、開源社群（Karakeep）。

---

## 7. 產品可能性評估

### 7.1 結論

| 定位 | 評估 |
|---|---|
| 大眾商業 App | ❌ **不建議重押**。紅海、免費替代品多、台灣市場小、付費意願低，基本功能 Raindrop、Karakeep 都已經做到 |
| 個人工具 | ✅ **非常值得**。技術範圍明確，一個週末就能做出可用的 MVP，而且能完全照自己的習慣設計 |
| 開源專案 / 作品集 | ✅ 適合。Android 原生分享、local-first、LLM 分類 pipeline 都是好題材 |
| 小眾商業楔子 | 🔶 **值得低成本驗證**：「繁中社群內容可靠擷取 + 主動回顧 + Bot 入口」 |

### 7.2 可能的差異化（依價值排序）

1. **主動回顧**（直接對準「存了不看」）：每週摘要推播、間隔重現、「這週存了 23 篇，只讀了 2 篇」。這比分類更有價值，也更難做好。
2. **取代「傳給自己」**：LINE Keep 結束後留下了斷層。LINE/Telegram Bot 入口零安裝，使用者的習慣完全不用改。
3. **在地社群內容**：Threads（台灣使用率很高）、PTT、Dcard、LINE 貼文。通用工具抓不到這些內容，是痛點也是門檻。但平台一改版就會壞，維運負擔很重。
4. 隱私 / local-first：有人在意，但這群人多半已經在用 Karakeep。
5. 自動摘要：已經是標配，不構成差異。

### 7.3 低成本驗證步驟（投入開發之前）

1. **先用 Raindrop 或 Karakeep 兩週當基準**，記錄：哪些連結抓不到標題、哪些分類常錯、存了之後重新打開的比例。
2. 做一個 **Telegram Bot**（比 LINE 免費、API 簡單）：只做「存連結 + 抓標題 + 每週摘要」，週末可以完成。給 10–20 位朋友用一個月。
3. 看兩個指標：**4 週留存**、**回顧推播的點擊率**。
4. 訪談 5–10 位目前「傳給自己」的人，問他們為什麼不用現成的 App。
5. 指標好再考慮 LINE Bot（⚠️ 先查 LINE 官方帳號的訊息費用方案）和上架。

---

## 8. 建議技術選型

| 層 | Phase 1：自用 MVP | Phase 2：AI + 同步 | Phase 3：產品化 |
|---|---|---|---|
| 客戶端 | Kotlin + Jetpack Compose | 同左，邏輯抽成純 Kotlin 模組 | KMP 共用邏輯 + iOS SwiftUI Share Extension |
| 入口 | `ACTION_SEND` 半透明 Activity + 分類 Direct Share 捷徑 | + Quick Settings Tile | + Telegram / LINE Bot |
| 本地儲存 | Room（SQLite）+ FTS4 全文搜尋 | 同左 | 同左 |
| 背景工作 | WorkManager | 同左 | 同左 |
| 擷取 | OkHttp + Jsoup（OG / JSON-LD）+ YouTube/X oEmbed | + Readability 全文 | + server 端補抓文章類網站 |
| 分類 | 使用者選 + 規則 | + 雲端 LLM（Haiku 級，結構化輸出） | + 裝置端 embedding kNN 擋掉大部分請求 |
| 後端 | **無** | Cloudflare Worker（LLM proxy）；要同步就加 Supabase | Workers / Cloud Run + Postgres（pgvector）+ 排程 |
| 回顧 | 本地通知（每週） | 同左 | 推播 / email 摘要 |

### Phase 1 實作清單（約 1–2 個週末）

- [ ] `ShareReceiverActivity`（半透明）：解析 `EXTRA_TEXT`，拆出 URL 和文字，寫入 Room，顯示 Toast，然後 `finish()`
- [ ] URL 正規化 + 去重（附單元測試，用各平台真實的分享字串當 fixture）
- [ ] `FetchMetadataWorker`：追 redirect、用 Jsoup 解析 OG/JSON-LD、YouTube oEmbed；失敗時標記 `blocked`
- [ ] 分類 CRUD + 每個分類發布 sharing shortcut（取前 N 個常用的）
- [ ] 網域規則（內建 youtube→影片、github→程式等，可讓使用者編輯）
- [ ] 主畫面：分類 tab + 未讀 / 已讀 + 搜尋 + 滑動封存
- [ ] 每週本地通知：「本週存了 N 篇未讀」

---

## 9. 待實測 / 未驗證清單

| # | 項目 | 方法 |
|---|---|---|
| 1 | 分類 shortcut 在 Motorola / Android 14–16 上會不會出現在 Sharesheet 頂端那排、能發布幾個 | 寫最小 demo，裝到實機測 |
| 2 | Sharesheet「固定 App」功能在目前這台手機上是否可用 | 實機長按試試看 |
| 3 | 各來源 App（Threads、IG、FB、X、LINE、PTT App、Dcard、小紅書）分享時 `EXTRA_TEXT` / `EXTRA_SUBJECT` 的實際格式 | 寫 debug 版把 intent 內容 dump 出來，收集 fixture |
| 4 | 各平台從手機端抓 OG 的成功率 | 用 fixture 批次跑 |
| 5 | Threads 有沒有公開 oEmbed | 實測 |
| 6 | Gemini Nano / ML Kit GenAI 在你的機型上能不能用 | `getBaseModelName()` |
| 7 | 各家 LLM 當期價格與中文 token 數 | 查官方定價頁，送 100 筆實際資料實測 |
| 8 | `multilingual-e5-small` 中文分類準確度與 APK 體積影響 | 用 Phase 1 累積的資料離線評估 |
| 9 | iOS extension 記憶體上限、免費帳號對 App Group 的限制 | 等做 iOS 時再查 |
| 10 | LINE 官方帳號訊息費用 | 等做 LINE Bot 時再查 |

---

## 10. 參考來源

- Android — Receive simple data: https://developer.android.com/training/sharing/receive
- Android — Direct Share targets: https://developer.android.com/training/sharing/direct-share-targets
- Android 11 Sharesheet pinning: https://www.androidauthority.com/android-11-share-sheet-app-pinning-1085350
- Apple — Share extension: https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/Share.html
- Web Share Target 支援度: https://caniuse.com/mdn-manifests_webapp_share_target ／ WebKit bug: https://bugs.webkit.org/show_bug.cgi?id=194593
- Flutter `receive_sharing_intent`: https://pub.dev/packages/receive_sharing_intent ／ `expo-share-intent`: https://github.com/achorein/expo-share-intent
- ML Kit GenAI: https://developers.google.com/ml-kit/genai ／ Gemini Nano: https://developer.android.com/ai/gemini-nano
- Anthropic pricing: https://platform.claude.com/docs/en/about-claude/pricing
- Pocket 關閉: https://techcrunch.com/2025/05/22/mozilla-is-shutting-down-read-it-later-app-pocket
- Omnivore 關閉: https://heise.de/-9998733
- LINE Keep 結束: https://www.bnext.com.tw/article/79030/line-keep-save
- Read-later 比較（2026）: https://wallabag.it/en/blog/2026/06/15/which-read-it-later-app-to-choose-in-2026/
- Raindrop — Pocket alternatives: https://blog.raindrop.io/pocket-alternatives/

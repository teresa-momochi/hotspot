# Interactive Image App — MVP 原型架構

> **這份 HTML 是「原型驗證」（Prototype for Validation），不是正式產品架構。**
> 正式版本是多人使用的雲端 App，需要真正的後端、資料庫、身分驗證與媒體儲存服務；
> 那些由 Database / Storage / Authentication / API / Media Specification 定義完成後，
> 才會有真正的正式架構與技術選型（是否维持 vanilla JS、是否需要框架、後端語言等，
> 都留到那時候依規格與團隊狀況決定，這裡不預先假設）。
>
> 這次的目的，是先把**畫面、導覽、互動、資料邊界**依已完成的 8 份規格驗證一遍，
> 並確保待補的規格完成後，能直接接上，而不必打掉重做。

依據已完成的 8 份規格建立，版本對應如下：

| Specification | Version | 本次採用重點 |
|---|---|---|
| 產品憲章 Product Constitution | 1.0 | 第14/15條：資料屬於使用者、預設安全可復原 |
| UX Design Principles | 1.0 | UX-01~24 全部套用 |
| Naming Specification | 1.0 | 正式名稱表、禁用詞清單 |
| International UI Specification | 1.0（重新定義） | English 為正式語言，教材內容不翻譯 |
| Navigation Specification | 1.0 | Welcome→EntryMethod→Login→Library→Project→ImageEditor |
| Screen Specification | 3.0 | Screen 組成（Header/Workspace/Panel/Overlay/Dialog）、Reading Mode 規格 |
| Interaction Specification | 2.0 | Drag Priority / Hotspot Priority |
| Data Specification | — | 概念模型 User→Project→Image→Hotspot→Content（僅概念，非 Schema）|

## 這次改了什麼：讓「不能只跑在單一 HTML」這件事真的成立

單一 HTML 檔本身沒問題（原型驗證階段很合適，跑起來快、方便你自己測），
真正的風險是**程式的邊界**如果假設了「同步、單分頁、記憶體內」，之後要換成
「非同步、多人、雲端」的時候，就會被迫整層重寫。所以這次把資料存取邊界拆成兩層：

```
Screens（畫面：Library / Project / Image Editor / Reading Mode ...）
        │  只呼叫 Repository，完全不知道底層是什麼
        ▼
Repository（async 介面；User → Project → Image → Hotspot 的所有操作都回傳 Promise）
        │  這一層的方法簽章 = 未來真正 API 應該長的樣子
        ▼
LocalState（目前唯一的實作：單分頁、記憶體內、同步運算 —— 這是「原型限定」的細節）
```

**關鍵：所有畫面程式碼都是 `await Repository.xxx()`，沒有任何地方直接碰 `LocalState`。**
這代表 Database／API／Storage／Authentication Specification 完成、要接上真正的多人雲端後端時：

- 只需要把 `LocalState` 換成真正呼叫後端的程式碼（`fetch`／SDK／Supabase 之類，屆時依規格決定）。
- `Repository` 的方法名稱與參數不需要變，因為畫面本來就是把它當「未來的 API」在呼叫。
- 每個 Repository 方法上都有 `TODO(對應規格)` 註解，標出正式版本這裡該補什麼（見下方清單）。

也因為畫面已經是 `async/await` 撰寫，本來就在處理「資料需要等待才會回來」這件事
（例如 Library、Project、Image Editor 進畫面時會先顯示「載入中」再补上內容），
不是假設資料永遠瞬間就緒——這正是多人雲端環境（有網路延遲）會遇到的情況，原型先把這個形狀跑順。

編輯互動點文字這種高頻操作，採用**樂觀更新（Optimistic Update）**：畫面先立即反應，
寫入動作在背後送出，不等它回來才更新畫面。這也是多人雲端 App 常見的正確模式
（若之後要做「多人同時編輯同一張圖」的衝突處理，會建立在這個模式上，而不是回頭改成處處等待）。

## 刻意仍未做的事

- **Media Specification** 尚未完成 → 圖片仍只用瀏覽器原生 `<input type=file>` 讀成暫時性的 Object URL，不落地儲存、不做壓縮/格式轉換/CDN。
- **Database Specification** 尚未完成 → `LocalState` 沒有資料表、欄位、型別設計，只是滿足 Repository 介面的最小示範實作。
- **API Specification** 尚未完成 → 沒有任何實際網路請求；`Repository` 的方法簽章是「未來 API 應該長什麼樣」的占位，不是正式合約。
- **Storage Specification** 尚未完成 → 沒有雲端同步、沒有登入 Session 持久化，重新整理頁面資料會消失（這是刻意的：避免我先自行決定要用 localStorage／IndexedDB／Supabase 等方式）。也因此 `checkSession()` 目前一定回傳「無 Session」，App 啟動一律要求重新登入；真正的 Session 持久化（例如 Supabase 的 JWT／refresh token）待 Authentication／Storage Specification 定案後才實作。
- **Authentication Specification** 尚未完成 → `Repository.login()` 的 Google／Apple／Email 三種方式都只是把假的 `currentUser` 放進記憶體，沒有真正的驗證。

所有邊界在程式碼中都用 `// TODO(Spec 名稱)` 註解標出。

## Login Flow / Session / Storage Usage 架構（本次新增）

依「教材製作工具，不是教材分享平台」的產品定位，這次把整體登入與資料隔離架構定義清楚：

```
App 啟動
   │
   ▼
Repository.checkSession()  ──有效──▶ 直接進 Library
   │ 無效（MVP 階段一律無效，因為沒有持久化 Session）
   ▼
Welcome → Entry Method → Login
   │
   ▼
Repository.login()          Authenticate User
   │
   ▼
Repository.getCurrentUser() Read User Profile（含 Plan）
   │
   ▼
Repository.getStorageUsage() Read Storage Usage
   │
   ▼
Open Library
```

這個順序已經是最終架構的樣子——`checkSession()` 內部從「一律回傳無 Session」換成「真正檢查 Supabase Session」時，Router 端完全不用改。

**資料隔離（Security）**：每個 Project 建立時都會綁定 `ownerId`（=目前登入使用者的 id）；
`listProjects`／`getProject`／`deleteProject` 一律先過濾 `ownerId === 目前使用者`，不是自己的 Project
會直接視為不存在。Image／Hotspot 不需要各自存 `ownerId`——擁有權透過 Project 繼承（User owns
Project owns Image owns Hotspot，與 Data Specification 一致）。目前用記憶體陣列 `filter()` 模擬，
之後接 Supabase 時換成 Row Level Security（RLS）比對 `auth.uid()`，畫面完全不用改。

**Session Expired Handling**：`AUTH_REQUIRED_SCREENS`（Library／Project／Image Editor／Reading／
Settings）在 `render()` 進場前會檢查 `LocalState.currentUser`，沒有登入使用者一律導回 Login，
不會讓畫面在缺少使用者資料的情況下嘗試渲染。

**User Menu**：Settings 畫面擴充為「我的帳戶（姓名／Email／登入方式／方案）＋ 儲存空間（用量長條，
示意資料，待 Storage Specification 定義真正計算方式）＋ 一般設定（進入方式／語言）＋ 登出」，沒有
新增畫面（Screen List 仍是 8 個），只是把既有的 Settings 內容補齊成完整的 User Menu 結構。

## 檔案結構

```
interactive-image-app-mvp.html   ← 原型驗證專用，正式版本不會是這個檔案
├── <style>      設計 Token（顏色／字級／間距）＋ 8 個畫面共用元件樣式
└── <script>
    ├── LocalState        原型限定：單分頁、記憶體內的資料
    ├── Repository        對外唯一介面，全部 async，未來替換內部即可，簽章不變
    ├── Router            畫面切換（依 Navigation Specification 的固定層級）
    ├── UI helpers         Toast／Dialog／EmptyState／Loading（UX-08, UX-09, UX-12, UX-13）
    └── Screens            SCR-001 ~ SCR-008，各自一個 render 函式，只呼叫 Repository
```

## 畫面清單與職責（對應 Screen Specification 第10節）

| 編號 | 畫面 | 角色（Navigation Roles 建議）| 主要工作 |
|---|---|---|---|
| SCR-001 | Welcome | 引導型 | 介紹產品，可 Skip |
| SCR-002 | Entry Method | 引導型 | 選擇加入書籤／主畫面／網址 |
| SCR-003 | Login | 驗證型 | 假登入（Google／Email 佔位）|
| SCR-004 | Library（學習庫）| 工作型 | 管理專案 |
| SCR-005 | Project（專案）| 工作型 | 管理圖片 |
| SCR-006 | Image Editor（互動設定）| 工作型 | 建立/編輯互動點 |
| SCR-007 | Reading Mode（閱讀模式）| 工作型 | 沉浸式體驗教材 |
| SCR-008 | Settings（設定）| 工作型 | 重設進入方式、切換介面語言 |

## 資料模型（僅概念，對應 Data Specification）

```
User
 └── Project        (擁有者: User)
      └── Image      (擁有者: Project)
           └── Hotspot   (擁有者: Image)
                └── Content  (文字說明；發音留待 Media Spec)
```

Ownership／Lifecycle 皆依 Data Specification：Create/Read/Update/Delete，刪除採軟刪除＋Undo Toast（呼應憲章第15條「預設安全，允許復原」），非做出一個獨立的「最近刪除」畫面（Screen List 目前 8 個畫面中未定義該畫面，先不新增）。

## 命名檢查（對照 Naming / International UI Specification）

- 畫面正式名稱、按鈕命名、提示語一律使用規範內用詞（新增圖片／刪除專案／找不到符合的專案 + 動作按鈕…）。
- 系統介面一律 `English（本地語言）` 並列顯示；使用者輸入的專案名稱／圖片名稱／互動點文字**不做任何翻譯或改寫**。
- 未使用清單中禁止詞（Asset／Workspace／Wizard／API／JSON...）於使用者可見文字。

## 之後怎麼接續（正式多人雲端架構）

1. Database／Storage／API／Authentication Specification 完成後，先設計正式的後端與資料表結構——這部分完全獨立於這份原型，不受它綁架。
2. 把 `LocalState` 換成真正的後端呼叫，`Repository` 介面簽章不變；畫面（Screens）與導覽（Router）邏輯不需要更動。
3. 在 `Repository` 補上使用者權限驗證與資料隔離（目前原型完全沒有，是正式上線前的必要項目）。
4. 屆時再視團隊與規格情況決定：是否仍維持 vanilla JS 單檔、拆成多檔案、或改用框架——這個決定留給那個階段，不在這次預先假設。


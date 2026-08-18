# Slate

> 一塊乾淨嘅寫字板 · 手機同電腦即時同步

**Live:** <https://backstudiobs.github.io/slate/>

一個純文字、跨裝置即時同步嘅筆記軟件。冇 markdown、冇 rich text、冇隱藏語法 —— 就係一塊寫字板，加分頁。

---

## 特色

- **純文字** —— 大 `<textarea>`，打乜見乜，冇任何 formatting 語法
- **手機／電腦即時同步** —— 停手打字 1.5 秒自動存，另一部機大約 1–2 秒後見到（最壞 8 秒 fallback poll）
- **分頁** —— 側邊 sidebar，可以新增／改名／刪除；tap 已選中嘅分頁即可 inline 改名
- **Google 登入** —— 用 Google Identity Services（唔係 Firebase 舊嗰個 popup／redirect），iOS Safari 都登得入
- **Email whitelist** —— 只有指定 email 可以用，兩層保護（app 層 + Firestore rule 層）
- **免費 hosting** —— GitHub Pages + Firebase Spark（免費 tier）
- **一個 HTML 檔** —— 冇 build step，冇 npm，冇 framework。所有 code 喺 `index.html` 入面

---

## 架構

```
                                     ┌──────────────────────────┐
                                     │  GitHub Pages            │
                                     │  backstudiobs.github.io  │
                                     │  ├── index.html          │
   ┌───────┐   ┌────────┐            │  └── firebase-config.js  │
   │  PC   ├──►│Chrome/ ├────────────┤                          │
   └───────┘   │Safari  │            └──────────────────────────┘
               └────┬───┘                          │
   ┌───────┐   ┌────┴───┐                          ▼
   │Mobile ├──►│Safari  │            ┌──────────────────────────┐
   └───────┘   │(iOS)   ├────────────►│ Google Identity Services │
               └────┬───┘             │ (ID token, no redirect)  │
                    │                 └────────────┬─────────────┘
                    │                              │
                    │                              ▼
                    │                 ┌──────────────────────────┐
                    │                 │ Firebase Auth            │
                    │                 │ (signInWithCredential)   │
                    │                 └────────────┬─────────────┘
                    │                              │
                    ▼                              ▼
             ┌──────────────────────────────────────────┐
             │ Cloud Firestore                          │
             │ /notes/{userId}                          │
             │   { tabs: [{ id, name, content, ...}] }  │
             └──────────────────────────────────────────┘
```

**點解用 Google Identity Services，唔用 Firebase 內建 Google Sign-In？**
Firebase 內建嘅 `signInWithPopup` / `signInWithRedirect` 會經 `<project>.firebaseapp.com/__/auth/handler` 個 domain，iOS Safari 因為 ITP（Intelligent Tracking Prevention）阻擋跨網站 cookie，登完之後攞唔返個 auth session。GIS 直接喺 client 攞 Google ID token（唔涉及第二個 domain），然後 `signInWithCredential` 換 Firebase session，完全繞開 ITP 問題。

---

## 檔案結構

```
Slate/
├── index.html                    ← 個 app 本體（HTML + CSS + JS 一個檔）
├── firebase-config.js            ← Firebase project keys（可以 public commit）
├── firebase-config.example.js    ← 範例，畀 fork 嘅人參考
├── .gitignore                    ← 排除 node_modules 之類
├── README.md                     ← 呢個檔
└── CHANGELOG.md                  ← 版本紀錄
```

---

## Setup from scratch（如果 fork 咗要重新 setup）

### 需要嘅嘢
- Google 帳號一個
- GitHub 帳號一個
- Mac 或 Linux（Windows 都可，只係下面 shell command 要改）
- 約 15–20 分鐘

### Step 1 — Firebase project（5 分鐘）

1. 去 <https://console.firebase.google.com/> → **Add project**（Google Analytics 唔洗開）
2. 入到去撳 **「＋ 新增應用程式」** → 揀 **Web** (`</>`) → nickname 任意 → **唔好** tick Firebase Hosting → **Register app**
3. 出咗嘅 `firebaseConfig = { ... }` 段 code **抄低**（等下要）
4. 左邊 **Authentication → Get started → Sign-in method → Google → Enable** → 揀 support email → Save
5. 左邊 **Firestore Database → Create database** → 揀最近嘅 location（`asia-east1` 台灣 / `asia-east2` 香港）→ **Production mode** → Create
6. Firestore 頁面頂部 **規則** tab → 貼呢條 rule（**記得改埋 email whitelist**）：

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /notes/{userId} {
         allow read, write:
           if request.auth != null
           && request.auth.uid == userId
           && request.auth.token.email_verified == true
           && request.auth.token.email.lower() in [
             'your-email@gmail.com',
             'another@example.com'
           ];
       }
     }
   }
   ```

   撳 **發布 / Publish**

### Step 2 — Google OAuth authorized origin

呢步好重要，唔做嘅話 GIS 會出 `origin_mismatch` 錯誤。

1. 去 <https://console.cloud.google.com/apis/credentials> → 揀你 Firebase 個 project
2. 揾嗰個 **Web client (auto created by Google Service)** → 撳個名進去
3. **已授權的 JavaScript 來源 / Authorized JavaScript origins** → **+ 新增 URI**
4. 加你嘅 GitHub Pages domain，例如 `https://你github帳號.github.io`
   - ⚠️ **冇 trailing slash**（唔可以係 `https://.../`）
   - ⚠️ **要 `https://`**
5. **儲存 / SAVE**（頁面最底藍色掣）
6. **等 5 分鐘**（Google 話最多 5 分鐘先生效）

順便抄低嗰個 **Web Client ID**（樣：`629976xxxxxx-xxxxxx.apps.googleusercontent.com`），等下要貼落 `index.html`。

### Step 3 — 改 `firebase-config.js` 同 `index.html`

**`firebase-config.js`** — 將 Step 1.3 抄低嗰段 config 貼入去：

```js
export const firebaseConfig = {
  apiKey: "AIza...",
  authDomain: "your-project.firebaseapp.com",
  projectId: "your-project",
  storageBucket: "your-project.firebasestorage.app",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abc..."
};
```

> **關於 apiKey 安全**：Firebase apiKey **公開放上網係 OK 嘅**，佢唔係密碼，只係認證個 project。真正安全靠 Firestore rule + email whitelist。呢個係 Google 官方立場，可以放心 commit 上 public repo。

**`index.html`** — 頂部個 script 入面搵呢兩個 constant，改晒佢：

```js
const GSI_CLIENT_ID = '你嘅Web Client ID.apps.googleusercontent.com';
const ALLOWED_EMAILS = [
  'your-email@gmail.com',
  'another@example.com'
].map(e => e.toLowerCase());
```

⚠️ `ALLOWED_EMAILS` 要同 Firestore rule 入面嗰個 list 一樣。兩層都要改。

### Step 4 — GitHub Pages 部署

```bash
# 開個 empty repo 喺 https://github.com/new (Public)
cd path/to/Slate
git init -b main
git add index.html firebase-config.js README.md CHANGELOG.md .gitignore
git commit -m "Initial commit"
git remote add origin https://github.com/你github帳號/slate.git
git push -u origin main
```

然後：
1. Repo Settings → **Pages** → Source: **Deploy from a branch** → Branch: **main** / **/(root)** → Save
2. 等 30–60 秒，頂部會出網址

### Step 5 — 加 authorized domain 落 Firebase Auth

**注意**：呢步同 Step 2 唔一樣。Step 2 加 origin 落 Google OAuth，呢步加落 Firebase Authentication。

1. Firebase Console → **Authentication → Settings → 已授權網域 / Authorized domains**
2. **新增網域 / Add domain** → 打 `你github帳號.github.io`（**冇 https://，冇 path**）→ Add

完成！開條 GitHub Pages 網址就用得。

---

## 修改配置

### 加／減 email whitelist

**要兩處一齊改**，唔改齊會出鬼問題：

1. **`index.html`** 頂部個 `ALLOWED_EMAILS` array（app 層檢查）
2. **Firebase Firestore Rules**（database 層檢查）

### 換 branding

- Title、副標題喺 `index.html` 個 `#loginScreen` section
- Header title 喺 `#userName` element（登入前顯示 "Slate"，登入後顯示 user 個名）
- Colors 喺頂部 `<style>` tag（用嘅係 stone / warm gray）

### 加新分頁 features

所有 tab logic 喺 `renderTabs()` 同 `startInlineRename()` 兩個 function。

### 加自動 backup／export

`notesData.tabs` 係一個 array of `{ id, name, content, updatedAt }` 嘅 object，可以 `JSON.stringify` 落 download link 做 export。

---

## Deployment / 更新流程

改完 code 之後：

```bash
cd ~/AIProjects/Slate
git add -A
git commit -m "描述改咩"
git push
```

推完 30–60 秒 GitHub Pages 就 rebuild。用之前**hard reload**（Chrome/Safari：Cmd+Shift+R）先睇到最新版。

---

## Troubleshooting / 常見問題

### 「登入失敗：origin_mismatch」

Google Cloud Console 個 OAuth client 未加你嘅 GitHub Pages domain 落 Authorized JavaScript Origins。返 Setup **Step 2** 做。已加咗但仲係 error → 等 5 分鐘 + Safari 完全 close app 再開。

### 「登入失敗：auth/unauthorized-domain」

Firebase Auth 未加你嘅 GitHub Pages domain 落 Authorized Domains。返 Setup **Step 5** 做。

### 「你嘅 email 冇權限使用呢個 app」

你個 email 唔喺 `ALLOWED_EMAILS` list 度。改 `index.html` 加你個 email，記得 Firestore rule 都加。

### iOS Safari 撳登入冇反應／登入完轉返 login page

如果你用緊 Firebase 舊嘅 `signInWithPopup` / `signInWithRedirect`，換做 GIS（見架構解釋）。而家個版本已經用 GIS 應該冇呢問題。

### 手機 → PC sync 快，PC → 手機 sync 慢

iOS Safari throttle 咗 Firestore realtime listener。而家個版本已經加咗 8 秒 fallback poll + visibility/focus 觸發 refetch，最壞 8 秒內見到。如果覺得慢可以將 `setInterval(..., 8000)` 改細個數字（例如 3000）。

### Git commit / push 唔到，話 `.git/HEAD.lock` exists

之前 Terminal 有個 git 未完成留低 lock file。跑：

```bash
rm -f .git/HEAD.lock .git/index.lock .git/objects/*/tmp_obj_*
```

之後再 commit / push。

### 分頁改名冇反應

Tap 嘅要係**已經 highlight 咗嘅 active tab**，先會變 input mode。如果 tap 未 active 嘅分頁，會 switch 過去。

---

## Tech stack

- **Frontend**: 一個 HTML 檔，vanilla JS，冇 framework／build step
- **Auth**: Google Identity Services + Firebase Auth (`signInWithCredential`)
- **Database**: Cloud Firestore (realtime + 8s fallback poll)
- **Hosting**: GitHub Pages (免費)
- **Storage cost**: Firebase Spark tier — 每日 50k reads / 20k writes / 1GB storage 免費
- **Actual usage**: 1 PC + 1 mobile tab 開住 = 大約 21k reads/day（未計 write），有一半 buffer

---

## Limitations

- **只支援 plain text**：冇 markdown 渲染、冇圖、冇 attachment
- **冇 offline mode**：冇 network 就寫唔到（Firestore 有 IndexedDB persistence 可以加，但暫時未實作）
- **冇 conflict resolution**：兩部機同時改同一個分頁，遲寫嘅覆蓋早寫嘅（last-write-wins per document）—— 對於個人筆記通常唔係問題
- **每個 user 一個 document**：所有分頁塞喺一個 Firestore document 度。Firestore document max 1MB，即係總 note 內容加起嚟唔可以超過 ~1MB（大約 500,000 中文字，普通用完全夠）
- **冇搜尋**：如果分頁多，要自己 Ctrl+F browser 搜（未加全 tab 搜尋）

---

## 授權

MIT。fork、改、再 host 悉隨尊便。

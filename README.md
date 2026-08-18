# Slate

> 一塊乾淨嘅寫字板 · 手機同電腦即時同步

Slate 係一個簡單、純文字、可以喺手機同電腦之間即時同步嘅筆記軟件。

* **登入方式**：Google 帳號（Firebase Auth）
* **儲存**：Firestore（免費 quota 已經夠一個人用一世）
* **打字位**：純 textarea，冇 markdown、冇 rich text、冇隱藏語法
* **分頁**：側邊 sidebar，可以隨時新增／刪除
* **同步**：打字停 1.5 秒自動存，另一部機大約 1–2 秒後見到

---

## 檔案結構

```
├── index.html                    ← 個 app 本身
├── firebase-config.js            ← 你自己嘅 Firebase config（自己整）
├── firebase-config.example.js    ← 範例，畀你抄
└── README.md                     ← 呢個檔
```

**重要**：`firebase-config.js` 唔會 auto 生成，你要自己跟下面步驟整。

---

## 第一部分：整 Firebase project（大約 5 分鐘）

### 1. 開 project

1. 去 <https://console.firebase.google.com/>
2. 用 Google 帳號登入
3. 撳 **「Add project」/「新增專案」**
4. 名任意（例：`my-notes`）
5. Google Analytics 可以 disable（唔需要）
6. 撳 **Create project**

### 2. 開 Web app

1. 入到 project 之後，撳 **「</>」** 圖示（Web）
2. App nickname 任意（例：`notes-web`）
3. **唔好** tick「Firebase Hosting」
4. 撳 **Register app**
5. 會出現 `firebaseConfig = { ... }` 呢舊 code —— **抄低成舊嘢**

### 3. 開 Google 登入

1. 左邊 sidebar → **Build → Authentication**
2. 撳 **Get started**
3. 揀 **Google** → toggle **Enable** → 揀你嘅 support email → **Save**

### 4. 開 Firestore

1. 左邊 sidebar → **Build → Firestore Database**
2. 撳 **Create database**
3. Location 揀近你嘅（例：`asia-east1` 台灣，最近香港）
4. **Start in production mode**（我哋會自己寫 rule）
5. 撳 **Create**

### 5. 貼 security rule（好重要，唔貼會讀寫唔到）

1. Firestore 個 tab 頂部 → **Rules**
2. **成個** paste 落去換走原本嘅：

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /notes/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

3. 撳 **Publish**

呢條 rule 嘅意思：每個 user 只可以讀寫自己 UID 嘅 document，其他人完全冇 access。

### 6. 將 config 貼入 `firebase-config.js`

1. 將 `firebase-config.example.js` copy 一份，改名做 `firebase-config.js`
2. 打開嚟改，將第 2 步抄低嗰舊 `firebaseConfig` 貼落去，成個檔應該係咁：

```js
export const firebaseConfig = {
  apiKey: "AIza...你嘅key...",
  authDomain: "my-notes-xxxxx.firebaseapp.com",
  projectId: "my-notes-xxxxx",
  storageBucket: "my-notes-xxxxx.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abc123def456"
};
```

> **關於 apiKey 安全**：Firebase 嘅 `apiKey` **公開放上網係 OK** 嘅，佢唔係密碼，只係用嚟認住個 project。真正嘅安全靠上面第 5 步嗰條 Firestore rule。呢個係 Google 官方立場，可以放心 commit 上 public repo。

---

## 第二部分：部署上 GitHub Pages（大約 3 分鐘）

### 1. 開 repo

1. 去 <https://github.com/new>
2. Repository name：例如 `my-notes`
3. **Public**（GitHub Pages 免費 plan 需要 public）
4. **唔好** tick 任何嘢（README/gitignore/license）
5. 撳 **Create repository**

### 2. Upload 檔案

**最簡單方法**：直接喺 GitHub 網頁 drag & drop

1. 入到新 repo 個頁
2. 撳 **「uploading an existing file」**
3. 將 `index.html`、`firebase-config.js`、`README.md` 三個檔 drag 落去
4. 撳 **Commit changes**

（**唔好** upload `firebase-config.example.js`，或者 upload 都無所謂——反正真正個 config 冇秘密。）

### 3. 開 Pages

1. Repo 頂 → **Settings** → 左邊 **Pages**
2. **Source**：`Deploy from a branch`
3. **Branch**：`main`（folder：`/root`）
4. 撳 **Save**
5. 等 30 秒–1 分鐘，佢會俾條網址你，例如：
   `https://你嘅githubuser.github.io/my-notes/`

### 4. 加 authorized domain 落 Firebase（重要！）

Firebase 預設只信 `localhost` 同 `你project.firebaseapp.com`。要加你嘅 GitHub Pages domain：

1. Firebase Console → **Authentication → Settings → Authorized domains**
2. 撳 **Add domain**
3. 打 `你嘅githubuser.github.io`（**冇 https://，冇 path**）
4. 撳 **Add**

搞掂！開條 GitHub Pages 網址 → 撳「用 Google 登入」→ 就可以用喇。

---

## 用法

* **新增分頁**：左邊 sidebar 撳「＋」
* **改分頁名**：撳上面條 input
* **打字**：中間大嗰個位，純 plain text
* **刪除分頁**：hover 過個分頁名就會見到 ×
* **手機**：撳左上角三條線開 sidebar
* **同步**：停手打字 1.5 秒自動存，另一部機幾秒後 refresh 就見到

登入之後，喺任何裝置用**同一個 Google 帳號**登入，睇到嘅內容都係一樣。

---

## Troubleshooting

**「需要 Firebase 設定」個 screen 一直出現**
→ `firebase-config.js` 未整好，或者名打錯咗。confirm 檔名 exact，同埋 hard reload（Ctrl+Shift+R）

**「登入失敗：auth/unauthorized-domain」**
→ 上面第二部分第 4 步冇做，去 Firebase → Authentication → Settings → Authorized domains 加你個 `.github.io` domain

**「連線錯誤」或者「儲存失敗」**
→ 通常係 Firestore rule 未 publish。返去 Firebase → Firestore → Rules，confirm 條 rule 同上面第一部分第 5 步一樣，撳 Publish

**手機 Google 登入 popup 出唔到**
→ 個 app 喺手機會自動用 redirect 代替 popup，如果 Chrome 阻擋咗就要 allow 個 site

**想睇下 raw data**
→ Firebase Console → Firestore → `notes` collection → 揀你嘅 UID

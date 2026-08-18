# Changelog

All notable changes to Slate.

## [1.0.0] — 2026-08-18

初版正式可用。

### Added
- 純文字 textarea 編輯器（冇 markdown、冇 rich text）
- 側邊分頁 sidebar：新增／刪除／inline 改名（tap 已 active 嘅分頁 = 改名）
- Google 登入（用 Google Identity Services + Firebase `signInWithCredential`）
- Cloud Firestore 即時同步（打字停 1.5 秒自動存）
- Email whitelist（app 層 + Firestore rule 層兩層保護）
- iOS Safari 兼容：8 秒 fallback poll + visibility/focus 觸發 refetch，解決 realtime listener 被 throttle 嘅問題
- Mobile responsive：hamburger menu、touch-friendly 分頁 list
- Sync status indicator（未載入 / 打緊字 / 儲存中 / 已同步 / 錯誤）

### Architecture decisions
- **用 Google Identity Services 代替 Firebase 內建 `signInWithPopup/Redirect`**
  - 原因：Firebase 內建 flow 依賴 `<project>.firebaseapp.com/__/auth/handler` 呢個 domain，iOS Safari ITP 阻擋跨網站 cookie，登完攞唔返 session
  - GIS 直接 client-side 攞 ID token，然後 `signInWithCredential(GoogleAuthProvider.credential(idToken))` 換 Firebase session，繞開 ITP
- **一個 HTML 檔，冇 build step**
  - 用 ES module `import()` 直接由 gstatic CDN load Firebase SDK
  - 純 vanilla JS，冇 framework
  - GitHub Pages 直接 serve，冇需要 Node build
- **每個 user 一個 Firestore document**（`/notes/{uid}`）
  - 所有分頁 array 塞入去
  - 換來嘅簡單：single query、single subscription、atomic write
  - 代價：total notes ≤ 1MB Firestore document limit（~500k 中文字）
- **8 秒 backup poll**
  - iOS Safari 會 throttle 背景 WebSocket / long-poll，realtime listener 有時 stall
  - 靜態 poll 每 8 秒喺 tab visible 時，確保最壞 case 8 秒內 sync
  - Cost：一日 ~10.8k reads / tab，Spark tier 免費 quota 50k / day 有 buffer

### Iterations during initial build
- v0.1: Firebase `signInWithRedirect` on mobile — iOS Safari 登入完轉返 login page（ITP 問題）
- v0.2: 切 `signInWithPopup` — iOS Safari popup 靜靜地 fail
- v0.3: 重寫用 Google Identity Services + `signInWithCredential` — 手機終於登到入
- v0.4: Realtime listener 喺 iOS Safari 上 stall（PC → mobile sync 慢）— 加 visibility/focus/interval refetch 解決
- v1.0: 加 email whitelist + Firestore rule 加強

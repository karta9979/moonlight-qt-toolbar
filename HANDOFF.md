# 交接文件：在 Moonlight 串流畫面內建「懸浮工具列」

> 交接日期：2026-11-06
> 專案擁有者：karta9979（杰）
> 本文件給接手的 Claude Code 閱讀。先讀完整份，再開始動手。

---

## 1. 故事背景（為什麼要做這件事）

1. 擁有者用 **Sunshine（被連端）+ Moonlight 電腦版（連線端）+ Tailscale** 架了一套遠端打電動的環境，並寫成公開教學：
   - GitHub：`karta9979/sunshine-moonlight-guide`
2. 使用時常需要按 Moonlight 的組合鍵（Ctrl+Alt+Shift+某鍵），四個鍵很難按，所以先用 **AutoHotkey** 做了外掛式的懸浮按鈕（見 `reference/` 裡兩個 `.ahk`）：
   - `Moonlight按鍵.ahk`：可拖動的「豬寶」圖案，點開是功能選單。
   - `Moonlight工具列.ahk`：深色膠囊工具列（收合／全螢幕／切換螢幕／更多）。
3. 外掛式做法遇到的問題：
   - 要另外安裝 AutoHotkey，別人照教學用要多一步。
   - AutoHotkey 是巨集工具，**線上遊戲反作弊（例如 AVA 的錯誤碼 E019100B）會擋**。已經改成「跟著 Moonlight 開關」來閃避，但治標不治本。
   - 它只是浮在 Moonlight 視窗上面，**不是真的在串流畫面裡**；全螢幕時可能被蓋住；要靠「模擬按鍵」轉送，偶爾會漏收。
   - Moonlight 設定「擷取系統鍵盤快捷鍵＝一律」時，AHK 送的 Ctrl+Alt+Shift+F1/F2（Sunshine 切換螢幕）會失效。
4. 因此決定：**fork 官方 moonlight-qt，直接在串流畫面裡加入工具列**，做成擁有者自己的 Moonlight 版本。

---

## 2. 目標

在 Moonlight 電腦版的**串流畫面內**，加入一個可點擊的懸浮工具列，外觀參考 `reference/inspiration-toolbar.png` 與 `reference/toolbar_preview.png`：

- 深灰色、半透明、圓角膠囊形狀。
- 預設在畫面**上方中間**。
- 可**收合**成一顆小圓鈕，再點展開。
- 可**拖動**改變位置（第二階段，見下方分階段）。

### 2-1 按鈕與功能

| 按鈕 | 功能 | 實作方向（建議） |
|---|---|---|
| ‹ 收合 / › 展開 | 工具列縮成小圓鈕，不擋畫面 | 純 UI 狀態 |
| ⛶ 全螢幕 | 全螢幕 ↔ 視窗 | 呼叫既有 `performSpecialKeyCombo(KeyComboToggleFullScreen)` |
| ⇄ 切換螢幕 | 主機螢幕 1 ↔ 2 | 直接用 `LiSendKeyboardEvent2` 送 Ctrl+Alt+Shift+F1 / F2 給主機（Sunshine 官方切換螢幕快捷鍵）；**不要**走本機模擬按鍵 |
| ⋯ 更多 | 展開選單 | 見下表 |

「更多」選單內容（照 AHK 版本）：

| 選單項目 | 實作方向 |
|---|---|
| 切換螢幕 1 / 切換螢幕 2 | 同上，指定 F1 / F2 |
| 貼上我的文字 | 既有 `KeyComboPasteText` |
| 放開滑鼠 | 既有 `KeyComboUngrabInput` |
| 效能資訊 | 既有 `KeyComboToggleStatsOverlay` |
| 結束串流 | 既有 `KeyComboQuit` |
| 隱藏工具列 | 本次串流不再顯示 |

AHK 版的「複製（對方）」「聲音／音量」**先不做**（第三階段再評估）。

### 2-2 設定

- 在 Moonlight 設定頁加一個選項「**顯示串流工具列**」，**預設關閉**（不改變原本使用者的行為）。
- 加一個快捷鍵切換工具列顯示（建議 Ctrl+Alt+Shift+T，先確認沒被既有 KeyCombo 佔用）。

---

## 3. 程式結構線索（2026-11-06 從 upstream master `4113dd2` 看到的）

> 以下是交接前快速瀏覽的結果，**請自己再確認**，不要照單全收。

- 串流主迴圈與事件：`app/streaming/session.cpp`（`SDL_MOUSEBUTTONDOWN` 在約 2266 行轉給 `m_InputHandler->handleMouseButtonEvent`）。
- 輸入處理：`app/streaming/input/`
  - `input.h`：`enum KeyCombo`（Quit、UngrabInput、ToggleFullScreen、ToggleStatsOverlay、ToggleMouseMode、ToggleCursorHide、ToggleMinimize、PasteText、TogglePointerRegionLock、QuitAndExit、ToggleKeyboardGrab）與 `performSpecialKeyCombo()`。
  - `keyboard.cpp`：`performSpecialKeyCombo()` 實作、`LiSendKeyboardEvent2` 的用法。
  - `mouse.cpp`：`handleMouseButtonEvent()` — 工具列的點擊要在這裡（或更前面）攔截，**點在工具列上的事件不能送給主機**。
  - `abstouch.cpp` / `reltouch.cpp`：觸控（筆電觸控螢幕），第二階段再處理。
- 畫面疊加層：`app/streaming/video/overlaymanager.h/.cpp`
  - 目前只有 `OverlayDebug`（效能資訊）與 `OverlayStatusUpdate` 兩種，都是 SDL_ttf 畫的**文字** surface。
  - 各 renderer 在 `ffmpeg-renderers/` 裡各自畫疊加層（例如 `d3d11va.cpp` 的 `renderOverlay()`，約 847 行；位置判斷約 1185 行）。
  - 新增工具列最可能的做法：新增一個 `OverlayType`（例如 `OverlayToolbar`），用 SDL 畫出工具列的圖片 surface，再讓各 renderer 依指定位置畫出來。

### 3-1 Renderer 範圍

- **第一階段只需要 Windows 上預設會用到的 renderer 正常**（通常是 `d3d11va`；請確認 Windows 預設與備援路徑，例如 `plvk`、`sdlvid`、`dxva2`）。
- 其他平台（Linux 的 vaapi/drm/egl、macOS 的 vt_*）至少要**能編譯、不壞掉**；工具列在那些平台不顯示也可以接受。

---

## 4. 建置方式

### 4-1 推薦：用 fork 的 GitHub Actions 自動編譯

- upstream 已經有 `.github/workflows/build.yml`（push / pull_request 觸發）→ `build-win-mac.yml`，會在 `windows-2025` 上裝 Qt、跑 `setup-deps.ps1`、編譯，並用 `upload-artifact` 上傳 `Moonlight-Windows-x64-<版本>`。
- fork 的 Actions 預設是關閉的，擁有者需要在 fork 的 **Actions** 分頁按「I understand my workflows, go ahead and enable them」。
- 每次 push 後，擁有者到 Actions → 該次 run → Artifacts 下載 Windows x64 版測試。
- 如果 CI 太慢或只想跑 Windows，可以在 fork 裡調整 workflow，但**不要改到 upstream 的發佈流程邏輯**以外的地方。

### 4-2 本機編譯（擁有者電腦，選用）

README 寫的 Windows 需求：Qt 6.11 以上（安裝時選 **MSVC**，不支援 MinGW）、Visual Studio 2026 Community、7-Zip（做安裝檔才需要）。子模組要 `git submodule update --init --recursive`。

---

## 5. 分階段計畫

| 階段 | 內容 | 完成標準 |
|---|---|---|
| **0. 調查** | 讀懂 overlay、輸入、設定頁（QML）的結構；確認 Windows 預設 renderer；列出要改的檔案 | 給擁有者一份中文說明：打算改哪些檔、怎麼改、風險 |
| **1. 最小可用版** | 設定開關＋固定在上方中間的工具列；四個按鈕（收合、全螢幕、切換螢幕、更多）；點工具列不會把點擊送給主機 | CI 編出的 Windows 版，擁有者實際連 Sunshine 測試四個按鈕都能用 |
| **2. 好用版** | 「更多」選單；可拖動並記住位置；觸控螢幕點擊；滑鼠移到上方才顯示（可選） | 擁有者測試通過 |
| **3. 加分** | 串流音量／靜音；自訂圖示（例如擁有者的豬寶 `reference/pig.png`）；中英文翻譯字串 | 視擁有者需求 |

每個階段做完都要停下來，讓擁有者測試後再繼續。

---

## 6. 限制與注意事項

- **授權**：moonlight-qt 是 **GPL-3.0**。fork 必須維持 GPL、保留原授權與著作權聲明，修改後的原始碼要公開（fork 本來就是公開的）。
- **不要破壞原本行為**：設定預設關閉時，Moonlight 的行為要跟官方版完全一樣。
- **改動盡量集中、少碰既有程式**，方便之後同步 upstream 的更新（`git fetch upstream` + merge/rebase）。
- 工具列的點擊與拖動事件**一律不能送到主機**。
- 相對滑鼠模式（遊戲模式、游標被抓住）時，工具列可能點不到；第一階段可以只在「游標可見」時讓工具列可點，並在說明裡寫清楚。
- 不要加入任何會被反作弊視為注入或掛鉤的東西（這也是擺脫 AHK 的初衷）。
- 不要提交擁有者的個人資訊（Tailscale IP、email、電腦名稱）。

---

## 7. 和擁有者溝通的方式

- 一律用**繁體中文**。
- 擁有者是資工碩士生，懂程式但**不熟 C++/Qt 和 git 進階操作**；講解時要一步一步、照順序說（先講前因、再講做法、最後講結論），不要一開始就丟結論。
- 需要擁有者操作的事（開 Actions、下載 artifact、在自己電腦測試、Sunshine 設定），要寫成**編號步驟**，標出要點的按鈕名稱（英文介面附中文翻譯）。
- 引用英文錯誤訊息或文件時：**先翻譯，再整理重點**。
- 修改既有說明文字或程式時，標出**位置（檔案、行號）並並列「原本 → 修改後」**。
- 做出會影響使用方式的決定前（例如快捷鍵選哪個、工具列預設位置），先問擁有者。

---

## 8. 測試環境（擁有者）

- 連線端：Windows 11，Moonlight 電腦版（目前官方 6.2.0），常用**視窗化**模式、勾選「為遠端桌面而非遊戲最佳化滑鼠」。
- 被連端：Windows，Sunshine（LizardByte 官方 2026.914.x）、雙螢幕。
- 網路：同區網或 Tailscale。
- 手機端（不在本專案範圍）：Android 用阿西西版、iPhone 用官方版。

---

## 9. reference/ 資料夾

| 檔案 | 說明 |
|---|---|
| `inspiration-toolbar.png` | 擁有者想要的外觀參考（其他軟體的工具列截圖） |
| `toolbar_preview.png` | 依參考圖畫的示意：左邊展開、右邊收合 |
| `Moonlight工具列.ahk` | 膠囊工具列 AHK 版，**行為規格的參考**（按鈕、收合、拖動、選單位置） |
| `Moonlight按鍵.ahk` | 豬寶 AHK 版（含選單全部功能、音量控制的 Windows Core Audio 寫法） |
| `ahk-pig-menu-demo.jpg` | AHK 豬寶版實際使用畫面 |
| `pig.png` | 擁有者自己畫的豬寶（75×96 去背），第三階段自訂圖示可用；**版權屬擁有者** |

---

## 10. 第一步請做

1. 讀 `CLAUDE.md` 與本文件。
2. 確認 repo 狀態、子模組、upstream remote（沒有就加：`https://github.com/moonlight-stream/moonlight-qt.git`）。
3. 做「階段 0 調查」，**先不要改程式**，用中文回報計畫給擁有者確認。

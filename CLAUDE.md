# 淡海機廠調車無線電練習

讓司機員練習淡海機廠調車情境中與 OCC 之間無線電用語的手機網頁工具。

## 專案位置與部署

| 項目 | 位置 |
|------|------|
| 本機專案 | `C:\Users\kunpe\Ai_output\無線電練習\` |
| 主檔案 | `index.html`（單檔 SPA，純前端 HTML/CSS/JS） |
| QR code | `qrcode.png`（指向 Pages 網址） |
| GitHub Repo | https://github.com/kunpeto/lrt-depot-radio-practice |
| GitHub Pages | https://kunpeto.github.io/lrt-depot-radio-practice/ |
| 部署方式 | main 分支 root 目錄，推送後自動更新（約 1–2 分鐘） |

更新流程：改 `index.html` → `git add . && git commit -m "..." && git push`。

## 相關知識來源

參考 `C:\Users\kunpe\projects\shared-knowledge\` 下：
- `lrt-depot-operations.md`（機廠軌道佈設、TK 編碼、號誌/轉轍器、模式/頻道對照、OCC 呼叫點、洗車軌單向規則、供電分區 A/B）
- `lrt-driver-manual.md`（駕駛操作、行車模式、集電弓）
- `lrt-broadcast-comm.md`（無線電頻道配置）

## 介面 / 選擇器

- **機廠架空線狀態**：正常 / A 區斷電（TK201+TK202）/ B 區斷電（TK203+TK204）/ 全區斷電
- **車號**：自由輸入（預設 101）
- **起點**：駐車區（P1/1–P4/4）/ 維修工廠（TK101–TK107）/ 轉換軌（上行、下行）/ 洗車軌（TK301）
- **目的地**：同起點選項 + 正線（上行往 V11 一月台、下行往 V11 二月台）

轉換軌顯示名稱：`轉換軌上行`、`轉換軌下行`（現場習慣稱呼，不用 TK501/TK502）。

## 核心資料模型

### `AREAS`
大區→細項對照表，`park / maint / trans / wash / main`。

### 點（pts）序列
`buildPoints(fromA, fromD, toA, toD, power)` 產生報告點陣列，每點結構：
```js
{
  loc,        // 位置字串（用於「[loc]，[車號]呼叫OCC，...」開頭）
  report,     // 抵達時回報事件（不含位置前綴與 OVER）
  extra,      // 附加回報（雙端電池 / 電壓 800V）
  confirm,    // OCC 要求確認項目陣列
  nextMode,   // OCC 指示的下一段行車模式
  nextLoc,    // OCC 指示的下一段目的地
  nextDo,     // 抵達後動作（停妥回報 / 降弓並投入ESS模式回報 / …）
  isFinal,    // 最終停妥點（OCC 只收到不下指令）
  // 標記：
  _occOnlyPrep,       // 動車前準備兩階段開機
  _maintDepShed,      // 維修出廠→駐車場棚外（升弓+換端）
  _maintToParkShed,   // 維修→駐車場（直達），目標軌棚外
  _targetNoPower,     // 目標駐車軌斷電
  _beforeMaint,       // 即將進入維修棚外（投 ESS）
  _beforePark,        // 即將進入駐車場棚外
  _transit,           // 中繼點（非停妥目的）
  _needFlip,          // 此中繼點需換端
  _speedLimit         // 5km / 25km 速限牌
}
```

### 對話段 chunks
`makeChunks(trainId, pts)` 將相鄰兩點組成 3 行對話段：
1. 司機員呼叫：`[loc]，[車號]呼叫OCC，[report][extra]，OVER`
2. OCC 回應：`OCC收到，[車號]，[report]，請[車號]確認[confirm]，以[nextMode]行駛至[nextLoc]，[nextDo]回報。OVER`
3. 司機員複誦：`[車號]收到，確認[confirm]，以[nextMode]行駛至[nextLoc]，[nextDo]回報，通話完畢。`

UI 一次只顯示一段，點「繼續」展開下一段。

## 路徑規則（buildPoints 實作要點）

### 連通性
| 路徑 | 規則 |
|------|------|
| 駐車場 ↔ 轉換軌 | 雙向，OCC 隨機指派 TK501/502 |
| 駐車場 ↔ 維修工廠 | 雙向直達 |
| 駐車場 ↔ 駐車場（換軌）| 須經轉換軌折返 |
| 維修工廠 ↔ 轉換軌/正線 | 須經駐車場 P_/4 換端 |
| 維修工廠 ↔ 維修工廠（換軌）| 須經駐車場 P_/4 換端，**不經轉換軌** |
| 洗車軌 TK301 | 單向：轉換軌 → TK301 → 駐車場 |

### 7 個 OCC 固定呼叫點
起點 → 25km 速限牌 → 轉換軌 → 5km 速限牌 → 駐車廠棚外 → 維修工廠棚外 → 目的地

### 速限牌自動插入（`insertSpeedLimitPoints`）
- 轉換軌 → 棚外/P_/4：插入 5 公里速限牌（切洗車連結模式）
- P_/4/棚外 → 轉換軌：插入 25 公里速限牌（切機廠模式）

### 升弓 / 降弓規則（重要）
- **維修工廠整區無架空線** → 進入維修須降弓投 ESS、離開須到有電區才升弓
- **維修出發的升弓地點在駐車場棚外**（不在維修棚外）：
  - 維修 → 駐車場（有電軌）：目標軌棚外升弓
  - 維修 → 駐車場（斷電軌）：維持 ESS，**不升弓**
  - 維修 → 轉換軌/正線/維修：隨機挑有電駐車軌 TKxxx 棚外升弓+換端
- **駐車場斷電區**不可升弓；駛入斷電目標軌前須降弓並投入 ESS
- 開機時：P1/1（固定無電）、所在供電區斷電、維修工廠 → 皆以 ESS 模式開機（兩階段：動車前準備 → 雙端電池 80/80%）

### 換端規則
「換端與否 = 接下來方向是否改變」
- 駐車場 → 正線（出廠方向一致）：**不換端**
- 駐車場 → 駐車場/維修/洗車（折返進廠）：**換端**
- 維修 → 轉換軌/正線/維修：於駐車場 P_/4 棚外換端

## 對話用語規則

### 呼號與結尾
- 呼號：`101車`（非「車組」）
- 司機員呼叫結尾：`OVER`
- 複誦結尾：`通話完畢`

### OCC 確認項目（鄰軌，非臨軌）
依情境組合：`前方號誌開通`、`轉轍器方位正確`、`鄰軌無列車`、`軌道淨空`、`目標駐車位號牌`、`前方鐵門開啟`、`洗車機雙燈綠燈`。

### 號誌機規則（棚外/速限牌無號誌）
| 目的點 / 路段 | 前方號誌開通 |
|-------------|-------------|
| 5km / 25km 速限牌 | ✗ 無 |
| 駐車場棚外（中繼/停妥）| ✗ 無 |
| 駐車場棚外 → P_/y（棚外→內）| ✗ 無 |
| 維修棚外 → 維修內（棚外→內）| ✗ 無 |
| P_/y 內 → 外（內往外）| ✓ 有（出口號誌）|
| 維修內 → 駐車場棚外（內往外）| ✓ 有（維修棚外出口號誌）|
| 轉換軌 approach | ✓ 有 |
| 駐車場棚外 → 維修棚外 | ✓ 有（維修棚外入口號誌）|

### 具體回報附加項
- ESS 模式：`雙端電池(80/80%)`
- 升弓：`電壓800V`

### 兩階段開機（維修、P1/1、斷電區）
1. 司機員：`...已完成車外車底巡視，將進入列車開機投入ESS模式，做好動車前準備後回報，OVER`
2. OCC：`...請[車號]完成動車前準備回報。通話完畢`
3. 司機員（複誦）→ 執行開機 →
4. 司機員：`...已開機並完成動車前準備，雙端電池(80/80%)，OVER`

## 號誌 / 轉轍器命名

- 號誌機：`I` 系列（現場/對話）；行控 ITMS 畫面顯示 `ESS`
- 轉轍器：`M` 系列（現場）；ITMS 畫面顯示 `EPM`
- 對話中通常省略不提具體編號

## 常見調整入口

| 想改 | 改哪裡 |
|------|-------|
| 新增/修改起訖點 | `AREAS` 物件（第 ~175 行） |
| 路徑合法性檢查 | `validate()` |
| 路徑連通邏輯 | `buildPoints()` 的 `directParkMaint` / `directMaintMaint` / `needTransferTrack` 等布林 |
| 升弓/降弓位置 | `buildPoints()` 內 `_maintDepShed` / `_maintToParkShed` / 進出棚外的 push |
| 速限牌插入 | `insertSpeedLimitPoints()` |
| OCC 確認項 / 下段指示 | `fillNextInstruction()` |
| 對話字串格式 | `makeChunks()` |
| UI 樣式 | `<style>` 區塊（暗色、手機 RWD） |
| 機廠斷電供電分區 | `isNoPower()` / `pickTransitParkTK()` / `pickTransitP4()` |

## QR code 重新生成

若 Pages 網址有變動：
```bash
curl -sL "https://api.qrserver.com/v1/create-qr-code/?size=500x500&data=<URL>&margin=20" -o qrcode.png
```

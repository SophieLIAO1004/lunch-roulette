# 今天吃什麼

抓附近正在營業的餐廳，丟進轉盤決定吃哪一家。純靜態 PWA，沒有後端、沒有資料庫。

---

## 一、拿到 API 金鑰（10 分鐘）

1. 進 [Google Cloud Console](https://console.cloud.google.com/)，建一個新專案。
2. 左側「API 和服務 → 程式庫」，搜尋 **Places API (New)**，按啟用。
   - 一定要是 **(New)** 這個。舊的 Places API 已列為 Legacy，新專案也啟用不了。
3. 「憑證 → 建立憑證 → API 金鑰」，複製出來。
4. 需要啟用帳單（要綁信用卡）。免費額度用完前不會扣款，但下一步一定要做。

## 二、鎖住金鑰跟花費（**別跳過**）

金鑰是寫在前端的，任何人都看得到。所以防護要做在 Google 那一端：

**限制 referrer** — 在金鑰的設定頁：
- 應用程式限制 → 網站
- 加入你的網域，例如 `https://你的名字.pages.dev/*`
- 本機測試再加 `http://localhost:*/*`

**限制 API** — 同一頁下方，API 限制選「限制金鑰」，只勾 Places API (New)。

**設配額上限** — 「API 和服務 → Places API (New) → 配額」，把每日請求上限設成一個你能接受的數字（例如 100）。這是唯一能真正保證不會收到帳單的機制。referrer 限制擋得住多數濫用，配額上限擋得住剩下的。

**設預算警示** — 「帳單 → 預算與快訊」，設一個 1 美元的預算通知。有任何一分錢被扣你都會收到信。

## 三、部署

### Cloudflare Pages
1. 把這個資料夾推上 GitHub
2. Cloudflare Dashboard → Workers & Pages → Create → Pages → 連結 repo
3. Build command 留空，Output directory 填 `/`
4. 部署完會拿到 `xxx.pages.dev`，把它加進上面的 referrer 限制

### GitHub Pages
1. 推上 GitHub → Settings → Pages → Source 選 `main` 分支的 `/ (root)`
2. 網址會是 `https://帳號.github.io/repo名/`

兩個都免費，也都自帶 HTTPS。**定位功能一定要 HTTPS 才會動**，所以不能用 `file://` 直接開檔案測試。本機測試用 `python3 -m http.server` 開 `http://localhost:8000`，localhost 是例外。

金鑰有兩種放法：讓每個人在 app 的「設定」裡自己貼（存在各自瀏覽器），或是直接寫進 `index.html` 的 `CONFIG.API_KEY`（大家共用一把，配額也共用）。20 個人的話後者方便很多。

## 四、建菜單資料

菜單沒有公開 API。Google 的 FoodMenus 屬於商家後台 API，只有店家自己能存取；外送平台也沒開放。所以這塊是手動維護的。

編輯 `index.html` 裡的 `MENUS` 陣列：

```js
const MENUS = [
  {
    match: ["八方雲集"],          // placeId 或店名關鍵字，任一命中就套用
    items: [
      { name: "招牌鍋貼 10 顆", price: 70 },
      { name: "酸辣湯餃 8 顆",  price: 75 },
    ],
  },
];
```

`match` 裡可以放店名關鍵字（比對顯示名稱有沒有包含），也可以放 placeId（精準，連鎖店的單一分店用這個）。搜尋結果每一列右邊的 **#** 按鈕會複製該店的 placeId。

有菜單的店會顯示「有菜單」標記，抽中後就多一顆「抽菜單」按鈕。

## 五、成本

用量都在瀏覽器裡算給你看（設定頁的「目前用量估算」）。

- 這個 app 只呼叫 **Text Search (New)**，一次搜尋 = 一次請求
- Text Search 屬 Pro 級距，每月前 5,000 次免費
- 欄位遮罩刻意排除了評分、照片、營業時間細節 —— 加進去會跳到 Enterprise 級距，免費額度只剩 1,000 次
- 交通時間是前端用直線距離推估的，**不呼叫任何 API**

20 人每天各用一次 ≈ 每月 600 次，離 5,000 還很遠。

想改用真實路網時間的話，接 Routes API 的 Route Matrix，但它按 element 計費（起點 × 終點），一次查 20 家就是 20 個 events，10,000 免費額度大概只夠 500 次查詢。以這個 app 的用途來說不太值得。

## 六、可以再加的東西

- **共享清單**：把 `MENUS` 跟收藏搬到 Cloudflare KV 或一份公開的 Google Sheet，大家就能共同編輯
- **多人一起劃**：現在是一支手機上劃掉不想吃的；要真的多人同步，需要一個 Workers + KV 的房間機制（仍在免費層內）
- **吃過的紀錄**：已經有了，最近 10 天內吃過的店在轉盤裡權重會降到 0.18，但不會完全排除

## 檔案

```
index.html            整個 app（樣式、邏輯、菜單資料都在裡面）
manifest.webmanifest  PWA 設定，讓它能加到主畫面
sw.js                 Service worker，快取 app 外殼；不快取 API 回應
icon.svg              圖示
```

iOS 加到主畫面的圖示偏好 PNG，想要更好看的話可以自己補一張 180×180 的 `apple-touch-icon.png`。

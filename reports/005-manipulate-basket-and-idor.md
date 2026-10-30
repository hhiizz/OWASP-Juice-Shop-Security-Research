# JS-005：★ ★ ★ Manipulate Basket 與額外 IDOR 漏洞研究

| 項目 | 內容 |
| --- | --- |
| 報告編號 | JS-005 |
| 報告建立日期 | 2026-10-30 |
| 實際測試日期／Juice Shop 版本 | 待補 |
| 測試環境 | OWASP Juice Shop，`http://localhost:3000` |
| 使用工具 | Burp Suite Repeater |
| 官方挑戰 | Manipulate Basket，★ ★ ★（3 星） |
| 官方分類 | Broken Access Control |
| 主要發現 | 重複 JSON 欄位造成授權檢查與寫入目標不一致 |
| 額外發現 | PUT 商品紀錄介面存在 IDOR／物件層級授權缺失 |
| 驗證狀態 | 依研究者實測：官方挑戰已完成；額外 IDOR 已以回應及資料庫前後比對確認 |
| 風險評估 | 待評估；三星為官方挑戰難度，不是額外 IDOR 的風險等級 |

## 1. 研究摘要

本次研究從新增購物車商品的 `POST /api/BasketItems/` 請求出發。直接將 `BasketId` 改為其他使用者的購物車時，伺服器會以 `401 Invalid BasketId` 拒絕；改用兩個同名的 JSON `BasketId` 欄位後，卻能通過檢查，將新商品加入其他使用者的購物車，完成 Manipulate Basket 挑戰。

接著檢查更新介面 `PUT /api/BasketItems/:id`，發現 Bender 帳號可以修改其他購物車的既有商品紀錄。伺服器回傳 `HTTP 200 OK` 與 `status: success`，資料庫前後比對亦顯示目標數量從 `2` 變為 `0`，構成另一項 IDOR 發現。

本文依研究者提供的請求、回應摘要與資料比對整理，未另行操作本機實驗環境。完整版本資訊可補入 [環境紀錄](../environment/README.md)。

## 2. 測試身分與物件關係

| 測試情境 | 操作帳號 | JWT 的 `bid` | 目標物件 | 目標所屬購物車 |
| --- | --- | --- | --- | --- |
| POST 重複欄位測試 | Admin | `1` | 新增 `ProductId = 3` | `BasketId = 3` |
| PUT 所有權測試 | Bender | `3` | `BasketItem.id = 4`、`ProductId = 4` | `BasketId = 2` |

`BasketItem.id` 是購物車商品紀錄的識別碼；`BasketId` 是該紀錄所屬購物車的識別碼，兩者不同。JWT 的 `bid` 用於理解本次登入身分與購物車的關係；伺服器仍需驗證 Token 並依可信的資料關聯進行授權。

以上數值來自本次實驗，其他部署的帳號與 ID 對應可能不同。

## 3. 主要挑戰：Manipulate Basket

### 3.1 挑戰目標

未經授權，將一項新商品加入其他使用者的購物車。官方說明指出，僅增加既有商品數量並不符合此挑戰要求。[官方 Broken Access Control 章節](https://pwning.owasp-juice.shop/companion-guide/latest/part2/broken-access-control.html)

### 3.2 直接修改 BasketId 遭到阻擋

使用 Burp Suite 擷取新增商品的 `POST /api/BasketItems/` 請求，並從登入取得的 JWT 了解目前帳號的 `bid`。本次 Admin 帳號的 `bid` 為 `1`。

當請求僅包含一個 `BasketId`，且直接改為其他使用者的購物車 ID 時，伺服器回傳 `401` 與 `Invalid BasketId`。這表示新增流程存在購物車檢查，單純替換 ID 並未成功。

### 3.3 使用重複 JSON 欄位繞過檢查

在 Burp Suite Repeater 中保留第一個 `BasketId` 為自己的購物車 `1`，另外加入第二個同名欄位，值設為目標購物車 `3`：

```http
POST /api/BasketItems/ HTTP/1.1
Host: localhost:3000
Authorization: Bearer <REDACTED>
Content-Type: application/json

{"ProductId":3,"BasketId":"1","quantity":1,"BasketId":"3"}
```

這裡的兩個 `BasketId` 必須以原始請求文字保留。若先轉為一般程式物件再序列化，重複欄位可能被合併，使測試內容改變。

### 3.4 測試結果

依研究者實測，該請求成功將商品加入其他使用者的購物車，並完成官方 Manipulate Basket 挑戰。

本次 POST PoC 使用 Admin 帳號；一般帳號在相同重複欄位路徑下的結果未另行提供。下節 PUT IDOR 則是使用 Bender 帳號測試。

### 3.5 成因分析：檢查與寫入使用不同值

本次行為符合 JSON 同名欄位處理不一致所造成的參數污染問題：

| 處理階段 | 使用的值 | 本次結果 |
| --- | --- | --- |
| 購物車授權檢查 | 第一個 `BasketId`：`1` | 與登入帳號的 `bid = 1` 相符 |
| 建立商品紀錄 | 最後一個 `BasketId`：`3` | 寫入其他使用者的購物車 |

官方 `v20.3.0` 的 `addBasketItem()` 會從原始本文收集欄位，使用 `basketIds[0]` 檢查購物車，再使用 `basketIds[basketIds.length - 1]` 建立紀錄，與本次觀察一致。[版本化原始碼](https://github.com/juice-shop/juice-shop/blob/v20.3.0/routes/basketItems.ts#L18-L45)

## 4. 額外發現：PUT 修改其他使用者的購物車商品

### 4.1 確認目標紀錄

測試前確認 `BasketItems` 中的目標紀錄如下：

| 欄位 | 修改前數值 |
| --- | --- |
| `id` | `4` |
| `BasketId` | `2` |
| `ProductId` | `4` |
| `quantity` | `2` |

目前登入的 Bender 帳號 JWT `bid = 3`，目標紀錄則屬於 `BasketId = 2`，顯示兩者不屬於同一個購物車。

### 4.2 使用 Bender 身分發送 PUT 請求

在 Repeater 中指定商品紀錄 ID `4`，只修改數量欄位：

```http
PUT /api/BasketItems/4 HTTP/1.1
Host: localhost:3000
Authorization: Bearer <REDACTED>
Content-Type: application/json

{"quantity":0}
```

請求路徑中的 `4` 是 `BasketItem.id`，並非目前帳號的 `bid`。請求沒有變更 `BasketId`，而是直接對他人的既有紀錄執行更新。

### 4.3 回應與資料庫前後比對

依實測紀錄，伺服器回傳 `HTTP 200 OK`，回應的 `status` 為 `success`。再次查詢相同紀錄，比對結果如下：

| 欄位 | 修改前 | 修改後 |
| --- | --- | --- |
| `BasketItem.id` | `4` | `4` |
| `BasketId` | `2` | `2` |
| `ProductId` | `4` | `4` |
| `quantity` | `2` | `0` |

前後比對顯示，修改發生在同一筆商品紀錄，購物車與商品關聯維持不變，只有數量從 `2` 變為 `0`。這比僅觀察成功回應更能支持實際寫入成立。

### 4.4 成因與分類

此結果顯示更新介面未有效阻止 Bender 修改不屬於其購物車的商品紀錄，符合 IDOR，也屬於 API 物件層級授權缺失（BOLA）。有效登入並不代表可以操作路徑 ID 指向的任何物件。[OWASP API1:2023](https://api-security.owasp.org/editions/2023/en/0xa1-broken-object-level-authorization/)

作為程式碼參考，官方 `v20.3.0` 的 `quantityCheckBeforeBasketItemUpdate()` 以路徑 ID 查找商品紀錄，但該函式中未比較紀錄所屬購物車與登入使用者的購物車；其數量判斷也以真假值決定是否執行數量檢查。[版本化原始碼](https://github.com/juice-shop/juice-shop/blob/v20.3.0/routes/basketItems.ts#L59-L79)

這份參考不能取代本機版本的完整路由分析。修正時必須檢查目標紀錄的所有權，不能只調整 `quantity = 0` 的驗證。

## 5. 影響與驗證範圍

| 發現 | 已驗證結果 | 驗證範圍 |
| --- | --- | --- |
| 重複 JSON 欄位繞過新增檢查 | 新商品加入其他購物車，官方挑戰完成 | Admin，`bid 1 → BasketId 3` |
| PUT 商品紀錄 IDOR | 他人商品數量從 `2` 改為 `0` | Bender，`bid 3 → BasketId 2`，紀錄 ID `4` |

已確認的影響為購物車資料完整性遭破壞：可新增非原使用者選擇的商品，或修改既有商品數量。是否影響結帳金額、訂單或其他帳號與商品紀錄，仍需額外測試；`quantity = 0` 也不等於已證明紀錄被刪除。

「額外發現」指本次研究在官方挑戰之外記錄的行為，不代表已確認為新的公開漏洞或取得 CVE。

## 6. 修補建議

1. **拒絕含重複名稱的 JSON 欄位。** 在欄位被一般物件解析合併前識別並拒絕歧義輸入；單純對已解析物件執行 schema 驗證，可能無法發現原始本文的重複欄位。
2. **讓授權與寫入使用同一個已驗證目標。** 新增商品時，由伺服器端可信身分與資料關聯取得購物車，不依賴使用者自行指定其他 `BasketId`。
3. **逐筆檢查商品紀錄所有權。** 更新時確認目標商品紀錄所屬購物車可由目前使用者操作；也可在更新條件中同時限制紀錄 ID 與已授權購物車，並確認受影響筆數。
4. **分開驗證權限與數量規則。** 不論數量是 `0`、正整數或其他輸入，都先執行所有權檢查，再依業務規則決定允許的數量或移除流程。

以上為修補建議，尚未實作或驗證。

## 7. 修補驗證計畫

修補後確認以下三點；目前尚未執行：

| 核心檢查 | 預期結果 |
| --- | --- |
| 重送含兩個 `BasketId` 的 POST 請求 | 拒絕請求，不新增商品 |
| 重送修改他人商品紀錄的 PUT 請求 | 拒絕修改，原數量不變 |
| 新增或修改自己購物車的商品 | 正常操作成功 |

## 8. 證據補充與研究心得

本報告已保留兩份遮蔽憑證的 PoC、失敗與成功回應摘要，以及 IDOR 的資料庫前後比對。可進一步補上 Juice Shop 版本、帳號與購物車關聯佐證、完整回應、挑戰完成畫面及查詢截圖。附件可放在 `assets/005-manipulate-basket-and-idor/`，從報告使用 `../assets/005-manipulate-basket-and-idor/<檔名>` 引用。

本次練習顯示，單一介面的 ID 檢查不代表所有購物車操作都受到保護。新增與更新介面必須各自驗證授權，且權限檢查的目標必須與實際寫入目標一致。搭配資料庫前後比對，能進一步確認成功回應是否對應到真正的跨使用者資料修改。

## 9. 參考資料

- [使用者提供的 Manipulate Basket 影片](https://youtu.be/pdtDtmIiSOQ)
- [官方 Broken Access Control 挑戰說明](https://pwning.owasp-juice.shop/companion-guide/latest/part2/broken-access-control.html)
- [Juice Shop v20.3.0：basketItems.ts](https://github.com/juice-shop/juice-shop/blob/v20.3.0/routes/basketItems.ts)
- [OWASP API1:2023 Broken Object Level Authorization](https://api-security.owasp.org/editions/2023/en/0xa1-broken-object-level-authorization/)

影片列為參考連結，未取得完整逐字稿核對。實測結果來自研究者提供的紀錄；原始碼引用用於成因說明，尚未與本機測試版本對齊。

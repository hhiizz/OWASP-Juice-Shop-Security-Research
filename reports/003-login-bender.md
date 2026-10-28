# JS-003：★ ★ ★ Login Bender

| 項目 | 內容 |
| --- | --- |
| 報告編號 | JS-003 |
| 報告建立日期 | 2026-10-28 |
| 挑戰難度 | ★ ★ ★（3 星） |
| 官方分類 | Injection |
| 漏洞類型 | SQL Injection |
| 目標 | 登入 Bender 的使用者帳號 |
| 使用工具 | Burp Suite、sqlmap；版本待補 |
| 驗證狀態 | 依研究者描述已成功完成 Bender 登入；原始請求、工具輸出與登入證據待補 |
| 測試日期／Juice Shop 版本 | 待補，參見 [環境紀錄](../environment/README.md) |
| 風險評估 | 待評估；三星代表挑戰難度 |

## 1. 漏洞摘要

本次透過 Burp Suite 擷取 HTTP 請求，將請求作為 sqlmap 的測試輸入，進行 SQL Injection 檢測與利用。依研究者描述，測試成功並完成 Bender 帳號登入。

SQL Injection 的核心問題是外部輸入可能影響原本的資料庫查詢語意。實際影響需依可注入的位置、查詢結構與資料庫權限確認。[OWASP SQL Injection 說明與防護](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

官方將 Login Bender 列為三星 Injection 挑戰，目標為登入 Bender 的帳號。[官方挑戰說明](https://pwning.owasp-juice.shop/companion-guide/latest/part2/injection.html)

## 2. 測試準備

- 確認瀏覽器請求會經過 Burp Suite，以便觀察並保存測試流量。
- 保存 HTTP 方法、路徑、標頭及請求本文，確保能重現相同輸入。
- 記錄 Juice Shop、Burp Suite 與 sqlmap 的版本。

目前未提供實際被測端點、參數名稱與 sqlmap 命令，因此本報告不補造特定注入位置、payload 或工具旗標。

## 3. 發現與利用流程

### 3.1 使用 Burp Suite 擷取請求

在實驗環境操作應用程式，使用 Burp Suite 擷取 HTTP 請求，觀察前端如何將輸入送至伺服器。保存目標請求，作為後續測試依據。

### 3.2 使用 sqlmap 測試 SQL Injection

將擷取的請求交由 sqlmap 進行檢測，分析輸入是否能影響後端 SQL 查詢，並利用檢測結果完成後續操作。

工具偵測結果應與原始回應、測試前後差異一同保存，才能說明注入點與利用效果。僅有工具名稱不足以重現漏洞。

### 3.3 完成 Bender 登入

依研究者描述，SQL Injection 利用成功，並完成 Bender 帳號登入。後續應保存登入後身分與挑戰狀態，確認取得的是指定目標帳號。

目前描述尚未區分「直接改變登入查詢以繞過驗證」與「先取得憑證，再以正常流程登入」兩種路徑。成功登入不等於已還原 Bender 的原始密碼，需以實際工具輸出與操作紀錄補充。

## 4. 重現流程整理

1. 在實驗環境使用 Burp Suite 擷取目標 HTTP 請求。
2. 保存完整請求，記錄待測參數與正常回應。
3. 使用 sqlmap 對該請求進行 SQL Injection 測試。
4. 依檢測結果進行利用，記錄實際 payload、回應差異與登入方式。
5. 確認登入後使用者為 Bender，保存挑戰完成或身分佐證。

上述流程整理自研究者描述；可直接重跑的原始請求與完整命令仍待補。

## 5. 結果與證據

| 項目 | 目前紀錄 |
| --- | --- |
| Burp Suite 擷取請求 | 研究者描述已執行；封包待補 |
| sqlmap SQL Injection 測試 | 研究者描述成功；命令、注入點與輸出待補 |
| Bender 帳號登入 | 研究者描述成功；登入身分或挑戰完成畫面待補 |
| 原始密碼還原 | 未提供，不宣稱已破解密碼 |

待補證據可放入 `assets/003-login-bender/`。發布封包與工具輸出前應遮蔽 Token、Cookie 等憑證。

## 6. 成因與影響分析

依 SQL Injection 的觀察結果，推測後端查詢未能將使用者輸入與 SQL 結構安全分離。未取得測試版本的程式碼前，尚不能確認具體查詢、函式或字串拼接位置。

依研究者描述，本次已達到登入 Bender 帳號的效果。是否能進一步讀取其他使用者資料、修改資料或取得更高權限，需另行驗證，不由單一帳號登入結果推論。

## 7. 修補建議

- 對受影響查詢使用參數化查詢，避免將輸入直接拼接到 SQL。
- 使用 ORM 時，同樣檢查 raw query 與動態查詢是否安全綁定參數。
- 依欄位需求驗證輸入格式，並限制應用程式資料庫帳號權限；這些措施作為參數化查詢以外的補強。[OWASP SQL Injection 防護建議](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

以上為修補建議，尚未實作或驗證。

## 8. 修補驗證計畫

| 測試情境 | 預期結果 | 實測 |
| --- | --- | --- |
| 重送原本成功的注入請求 | 無法改變查詢語意或取得未授權登入 | 未執行 |
| 使用錯誤密碼正常登入 | 驗證失敗，不取得目標身分 | 未執行 |
| 使用正確憑證正常登入 | 正常登入正確帳號 | 未執行 |

## 9. 參考資料與心得

- [使用者提供的 Login Bender 影片](https://youtu.be/a6kh9fL77A0)
- [官方 Injection 章節](https://pwning.owasp-juice.shop/companion-guide/latest/part2/injection.html)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

本次練習串接了 HTTP 請求觀察與 SQL Injection 自動化測試。完整報告應能把工具結果連結到具體注入點、應用程式回應與最終取得的使用者身分。

本文依研究者提供的流程與成功結果整理，影片列為參考連結，未取得完整逐字稿核對；未自行執行 Burp Suite 或 sqlmap 重跑測試。

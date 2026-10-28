# JS-004：★ ★ ★ Login Jim

| 項目 | 內容 |
| --- | --- |
| 報告編號 | JS-004 |
| 報告建立日期 | 2026-10-28 |
| 挑戰難度 | ★ ★ ★（3 星） |
| 官方分類 | Injection |
| 研究主題 | SQL Injection 取得帳號雜湊，搭配預先計算查找表嘗試還原密碼 |
| 目標 | 登入 Jim 的使用者帳號 |
| 使用工具／服務 | 既有 SQL Injection 測試流程、CrackStation |
| 驗證狀態 | 已記錄研究者描述的雜湊取得與查表嘗試；命中結果及登入證據待補 |
| 測試日期／Juice Shop 版本 | 待補，參見 [環境紀錄](../environment/README.md) |
| 風險評估 | 待評估；三星代表挑戰難度 |

## 1. 漏洞摘要

本次沿用先前發現的 SQL Injection 漏洞，查詢使用者資料與對應的密碼雜湊，再將 Jim 對應的雜湊交由 CrackStation 嘗試查找原始密碼。

依研究者描述，本次針對 MD5 雜湊進行破解嘗試。MD5 是雜湊函數，不是可透過金鑰解密的加密演算法；破解過程是尋找能產生相同雜湊值的候選明文，而不是執行 MD5 解密。[OWASP 密碼雜湊說明](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)

官方將 Login Jim 列為三星 Injection 挑戰。本報告記錄的是研究者採用的「SQL Injection → 憑證資料外洩 → 雜湊查表 → 登入驗證」路徑；最後的登入驗證仍待補結果。[官方挑戰說明](https://pwning.owasp-juice.shop/companion-guide/latest/part2/injection.html)

## 2. 前置條件

- 已掌握可查詢使用者資料的 SQL Injection 入口。
- 可辨識資料列中的帳號及其對應密碼雜湊，避免將不同使用者的資料混用。
- 了解本次取得雜湊的演算法與格式；若僅觀察到 32 位十六進位字串，仍需其他證據確認為 MD5。

相關工具使用流程可參考 [Login Bender 報告](003-login-bender.md)，但本次實際查詢端點、參數與 SQL 內容尚未提供，不能直接假定與 Bender 的請求完全相同。

## 3. 發現與利用流程

### 3.1 透過 SQL Injection 查詢使用者雜湊

沿用先前的 SQL Injection 漏洞，查詢使用者及其對應的密碼雜湊。從查詢結果中辨識 Jim 的帳號，取出屬於該帳號的雜湊資料。

此步驟的關鍵是維持「帳號與雜湊」的對應關係。僅取得一串雜湊，尚不足以確認該資料可用於登入 Jim。

### 3.2 使用 CrackStation 嘗試查表

將取得的實驗雜湊提交至 CrackStation，觀察是否能找到相符的候選密碼，並保存辨識類型與查找結果。

CrackStation 官方說明使用大型預先計算查找表，透過雜湊與候選密碼的對應關係進行搜尋，主要適用於未加鹽雜湊。因此本報告採用「預先計算查找表」描述；不將 CrackStation 的機制直接等同於特定的彩虹表實作。[CrackStation 運作說明](https://crackstation.net/)

若查找未命中，只表示服務目前未找到對應結果，不代表密碼無法被其他方法猜中。是否加鹽與確切儲存方式仍需由測試資料或對應版本程式碼確認。

### 3.3 驗證候選密碼與登入結果

若查找命中，應先確認候選明文計算出的 MD5 與取得的雜湊相符，再使用 Jim 的帳號及候選密碼進行正常登入，確認最終使用者身分。

這一步是完成驗證所需的後續步驟，目前尚未提供查找命中或登入成功的結果。

## 4. 重現流程整理

1. 重現既有 SQL Injection 漏洞，查詢使用者帳號與密碼雜湊。
2. 找到 Jim 的資料列，保存帳號與雜湊的對應證據。
3. 確認雜湊格式與演算法；本次研究者描述為 MD5。
4. 使用 CrackStation 對該實驗雜湊進行查找，記錄命中或未命中。
5. 若取得候選明文，先比對其雜湊，再嘗試登入並確認 Jim 的身分。

前四步依研究者描述整理；第五步的候選比對與登入結果仍待補。

## 5. 結果與證據

| 項目 | 目前紀錄 |
| --- | --- |
| SQL Injection 取得使用者與雜湊 | 研究者描述已取得；原始查詢及回應待補 |
| Jim 與目標雜湊的對應關係 | 需補查詢結果佐證 |
| MD5 類型 | 依研究者描述；辨識或程式碼佐證待補 |
| CrackStation 查找 | 研究者描述已嘗試；命中狀態與輸出待補 |
| Jim 登入與挑戰完成 | 尚未提供結果 |

待補證據可放入 `assets/004-login-jim/`。未提供的實際雜湊、明文密碼與 SQL 查詢不在報告中補造。

## 6. 成因與影響分析

此利用鏈包含兩個層次：首先，SQL Injection 讓使用者憑證資料暴露；其次，若密碼使用未加鹽 MD5 儲存，則可能透過預先計算查找表或候選密碼測試進一步還原。

一旦取得密碼雜湊，後續離線猜測不需要向登入端點逐次送出請求，因此登入限速本身無法保護已外洩的雜湊。若成功還原且密碼仍有效，便可能透過正常登入流程冒用帳號。

本次尚未提供破解命中或登入成功證據，其他帳號是否受到相同影響也需另行確認。

## 7. 修補建議

- 修正 SQL Injection：將使用者輸入與 SQL 結構分離，使用參數化查詢，並限制資料庫帳號權限。[OWASP SQL Injection 防護建議](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- 改善密碼儲存：以 Argon2id 等專為密碼設計的方案取代 MD5，使用每筆密碼獨立的 salt，並設定合適的運算成本。[OWASP 密碼儲存建議](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- 減少憑證暴露：API 回應避免包含密碼雜湊，檢查查詢結果序列化與敏感欄位處理。
- 若已確認憑證外洩，重設受影響密碼、撤銷相關工作階段，並檢查異常登入。

以上為修補建議，尚未實作或驗證。

## 8. 修補驗證計畫

| 測試情境 | 預期結果 | 實測 |
| --- | --- | --- |
| 重送原 SQL Injection 請求 | 無法取得未授權使用者資料 | 未執行 |
| 正常 API 查詢 | 回應不包含密碼雜湊 | 未執行 |
| 不同帳號設定相同測試密碼 | 使用獨立 salt 後，儲存結果不同 | 未執行 |
| 檢查新密碼的儲存格式 | 使用設定的密碼雜湊方案，並確認成本參數 | 未執行 |
| 使用已撤銷的憑證或工作階段 | 無法繼續存取帳號 | 未執行 |

## 9. 參考資料與心得

- [使用者提供的 Login Jim 影片](https://youtu.be/zJpJibswGWA)
- [官方 Injection 章節](https://pwning.owasp-juice.shop/companion-guide/latest/part2/injection.html)
- [CrackStation：How CrackStation Works](https://crackstation.net/)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)

本次練習連結 SQL Injection 與密碼儲存風險：雜湊不是明文，但外洩後仍可能被猜測或查表還原。完整驗證應同時保存資料取得、帳號對應、候選比對與登入結果。

本文依研究者提供的流程整理，影片列為參考連結，未取得完整逐字稿核對；未代為提交雜湊至 CrackStation，也未宣稱已取得未提供的破解結果。

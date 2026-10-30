# Juice Shop Security Research

以 OWASP Juice Shop 為練習平台，記錄漏洞重現、成因分析、影響評估與修補方法。

本專案為個人資安學習與研究紀錄。Juice Shop 是刻意包含漏洞的訓練應用程式；重現其既有挑戰不代表發現新的漏洞。

## 研究範圍

- 測試對象：自行部署或明確獲得授權的 OWASP Juice Shop 實驗環境。
- 研究內容：漏洞發現過程、可重現步驟、驗證證據、程式碼成因與修補建議。
- 每篇報告記錄實際測試的版本、前置條件與驗證限制。

## 報告索引

以下收錄研究報告與目前驗證狀態；各項發現的證據與限制請見報告內文。

| 編號 | 挑戰／漏洞名稱 | 挑戰難度 | 漏洞類型 | 風險評估 | 報告 |
| --- | --- | --- | --- | --- | --- |
| JS-001 | API-Only XSS | ★ ★ ★ | XSS／存取控制 | 待評估 | [報告：商品竄改已依實測描述記錄，XSS 證據待補](reports/001-api-only-xss.md) |
| JS-002 | Login Amy | ★ ★ ★ | Sensitive Data Exposure／規律密碼與字典攻擊 | 待評估 | [研究報告](reports/002-login-amy.md) |
| JS-003 | Login Bender | ★ ★ ★ | Injection／SQL Injection | 待評估 | [研究報告](reports/003-login-bender.md) |
| JS-004 | Login Jim | ★ ★ ★ | Injection／SQL Injection 與密碼雜湊破解 | 待評估 | [研究報告](reports/004-login-jim.md) |
| JS-005 | Manipulate Basket 與額外 IDOR | ★ ★ ★ | Broken Access Control／JSON 參數污染／IDOR | 待評估 | [研究報告：挑戰完成與跨購物車修改](reports/005-manipulate-basket-and-idor.md) |

挑戰難度依測試版本中 Juice Shop 的星級填寫，保留原始星數，不限定為一至四星。星級代表挑戰難度；風險評估需另外依據實際影響與利用條件說明。

## 目錄結構

```text
Juice-Shop-Security-Research/
├── README.md
├── environment/
│   └── README.md
├── reports/
│   ├── TEMPLATE.md
│   ├── 001-api-only-xss.md
│   ├── 002-login-amy.md
│   ├── 003-login-bender.md
│   ├── 004-login-jim.md
│   └── 005-manipulate-basket-and-idor.md
└── assets/
    └── README.md
```

所有報告統一放在 `reports/`，不建立分類子資料夾。漏洞類型與挑戰星級記錄在報告及上方索引中。

## 新增研究報告

1. 依照 [環境紀錄](environment/README.md) 填寫部署方式與版本。
2. 在 `reports/` 內複製 [漏洞報告範本](reports/TEMPLATE.md)，建立新報告。
3. 以 `001-short-description.md` 格式命名，編號在整個專案中保持唯一。
4. 將證據放入 `assets/001-short-description/`，並在報告中使用相對路徑引用。
5. 完成後更新上方報告索引。

未完成的測試或未驗證的修補，請明確標註狀態。風險評估如使用 CVSS，需附上版本、向量與評分理由。

## 參考資料

- [OWASP Juice Shop 官方專案](https://owasp.org/projects/juice-shop)
- [OWASP Juice Shop 原始碼](https://github.com/juice-shop/juice-shop)
- [官方學習指南](https://help.owasp-juice.shop/)

參考官方解答或其他研究時，請在各篇報告註明來源，並說明自己的驗證與分析。

# 報告證據

每篇報告使用獨立資料夾，名稱與報告檔名一致，例如：

```text
assets/
└── 001-short-description/
    ├── 01-request.png
    └── 02-result.png
```

- 使用能說明重現過程與結果的檔名。
- 在報告內解釋證據內容，不僅貼上截圖。
- 保留重現所需資訊，遮蔽真實密碼、Token、Cookie 與個人資料。
- 從 `reports/<漏洞類型>/` 引用時，使用 `../../assets/<報告名稱>/<檔名>`。

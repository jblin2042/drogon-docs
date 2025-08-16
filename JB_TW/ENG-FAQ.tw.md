
# 常見問題 FAQ

[原文：ENG-FAQ.md](/ENG/ENG-FAQ.md)

這裡列出一些常見問題及解答，並附有延伸說明。

## Drogon 的執行緒模型與最佳實踐是什麼？

Drogon 會在執行 `app().run()` 時建立 HTTP 伺服器執行緒與資料庫執行緒，這些執行緒組成一個執行緒池。Drogon 採用序列化任務處理系統。因此，建議盡量使用非同步 API 或協程來開發。詳細說明請參考：[深入理解 Drogon 執行緒模型](/JB_TW/ENG-FAQ-1-Understanding-drogon-threading-model.tw.md)。

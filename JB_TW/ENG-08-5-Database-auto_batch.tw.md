## 資料庫 - 自動批次模式

[原文：ENG-08-5-Database-auto_batch.md](/ENG/ENG-08-5-Database-auto_batch.md)

自動批次模式僅適用於 postgresql 14 以上版本的 client library，其他情況會被忽略。說明自動批次處理前，先介紹 pipeline 模式。

### pipeline 模式

自 postgresql 14 起，其 client library 提供 pipeline 模式介面。在 pipeline 模式下，可直接將新的 SQL 請求送至伺服器，無需等待前一請求結果回傳（與 HTTP pipeline 概念一致）。詳情請參考 [Pipeline mode](https://www.postgresql.org/docs/current/libpq-pipeline-mode.html)。此模式有助於效能提升，讓較少的資料庫連線可支撐更大量併發請求。

drogon 自 1.7.6 版起支援此模式，會自動檢查 libpq 是否支援 pipeline，若支援，透過 drogon DbClient 發送的所有請求皆採 pipeline 模式。

### 自動批次模式

預設下，非交易型 client，drogon 會為每個 SQL 請求建立同步點（synchronization point），即每個 SQL 都是隱式交易，確保 SQL 請求彼此獨立，讓 pipeline 與非 pipeline 模式對使用者而言完全等價。

但每個 SQL 都建立同步點會有效能負擔，因此 drogon 提供自動批次模式，改為每隔數個 SQL 才建立同步點。建立同步點的規則如下：

- EventLoop 迴圈中最後一個 SQL 必須建立同步點；
- 寫入資料庫的 SQL 後建立同步點；
- 大型 SQL 後建立同步點；
- 同一連線自上次同步點後連續 SQL 數達上限時建立同步點；

注意，同一連線兩個同步點間的 SQL 屬於同一隱式交易。drogon 不提供顯式開關同步點介面，因此這些 SQL 可能邏輯上無關，但因同屬一交易，會互相影響，故此模式並非完全安全，可能有以下問題：

- 失敗的 SQL 會導致前一同步點後的所有 SQL 回滾，但使用者不會收到通知（因未用顯式交易）；
- 失敗的 SQL 會導致下個同步點前的所有 SQL 都回傳失敗；
- 判斷寫入資料庫僅靠關鍵字比對（如 insert, update），無法涵蓋所有情境，例如 select 語句呼叫儲存程序，因此 drogon 雖盡力降低自動批次模式負面影響，但仍非完全安全；

因此，自動批次模式有助效能提升，但安全性不足，使用時請自行斟酌。例如僅用於純讀取 SQL。

> **注意** 即使純讀取 SQL 也可能導致交易失敗（如 select timeout），其後續 SQL 也會受影響而失敗（可能不符使用者預期，因邏輯上彼此無關），故嚴格來說，自動批次模式僅適用於讀取且非關鍵資料查詢。建議使用者為此類 SQL 另建自動批次 DbClient。當然，透過自動批次模式 DbClient 產生的交易物件則可安全使用。

### 啟用自動批次模式

使用 newPgClient 介面建立 client 時，第三個參數設為 true 即啟用自動批次模式；
使用設定檔建立 client 時，auto_batch 選項設為 true 即啟用自動批次模式。

## 下一步: [請求參考](/JB_TW/ENG-09-0-References-request.tw.md)

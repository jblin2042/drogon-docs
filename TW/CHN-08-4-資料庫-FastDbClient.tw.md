[English](/ENG/ENG-08-4-Database-FastDbClient)

# 資料庫 - FastDbClient

顧名思義，FastDbClient 提供比一般 DbClient 更高效能。與 DbClient 各自擁有 EventLoop 不同，FastDbClient 與 Web 應用的網路 IO 執行緒及主執行緒共用 EventLoop，內部可採無鎖設計，效能更佳。

實測下，極限高負載時 FastDbClient 比 DbClient 效能提升約 10%~20%。

## 建立與取得

FastDbClient 必須透過框架介面或設定檔建立。使用框架的 createDbClient 介面，最後一個參數設為 true 即建立 FastDbClient。

設定檔每個 db_client 項下有 is_fast 子選項，設為 true 即代表此物件為 FastDbClient。

框架會針對每個 IO 事件循環及主事件循環建立獨立 FastDbClient，每個 FastDbClient 內部管理多個資料庫連線。IO 事件循環數由 "threads_num" 控制，通常設為主機 CPU 核心數；每個事件循環管理的連線數由 "connection_number" 控制，因此一個 FastDbClient 的總連線數為 `(threads_num+1) * connection_number`，詳見[設定檔](/CHN/CHN-10-配置文件#db_clients)。

取得 FastDbClient 介面與一般 DbClient 類似：

```c++
orm::DbClientPtr getFastDbClient(const std::string &name = "default");
// 使用 drogon::app().getFastDbClient("clientName") 取得
```

需注意，因 FastDbClient 特性，必須在 IO 事件循環執行緒或主執行緒內呼叫上述介面，否則僅會取得空指標，無法使用。

## 使用方式

FastDbClient 用法與一般 DbClient 幾乎一致，但有以下限制（高效能需付出使用上的約束）：

* 取得與使用皆須在框架 IO 事件循環執行緒或主執行緒內，其他執行緒僅會取得空指標，若在其他執行緒使用將導致不可預期錯誤（因破壞無鎖條件）。好在大多數應用程式碼都在 IO 執行緒內，如控制器處理函式、過濾器函式等。FastDbClient 介面的各種回呼也都在 IO 執行緒，可放心嵌套使用。
* 絕不可使用 FastDbClient 的阻塞介面，因阻塞會卡住目前執行緒，而該執行緒同時負責資料庫 IO，將造成永久阻塞，無法取得結果。
* 同步交易建立介面可能阻塞（若所有連線皆忙），因此 FastDbClient 的同步交易建立介面直接回傳空指標。若需在 FastDbClient 上使用交易，請用非同步交易建立介面。
* 使用 FastDbClient 建立 ORM 的 Mapper 物件時，也僅能使用非同步非阻塞介面。

# 下一步：[自動批次處理](/CHN/CHN-08-5-数据库-自动批处理)

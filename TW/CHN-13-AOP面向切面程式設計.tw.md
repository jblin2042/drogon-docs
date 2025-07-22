[English](/ENG/ENG-13-AOP-Aspect-Oriented-Programming)

## 面向切面程式設計 - AOP

AOP（面向切面程式設計）是一種統一維護程式功能的技術。利用 AOP 可將業務邏輯各部分隔離，降低耦合度、提升重用性並加速開發效率。

受限於 C++ 語言特性，Drogon 未提供如 Spring 般靈活的 AOP，僅支援簡易 AOP，所有插入點皆內建於框架，使用者可透過 AOP 介面註冊特定處理程序（Advice）至插入點。

### 內建插入點

Drogon 提供 7 個插入點，應用程式執行至插入點時會依序呼叫使用者註冊的 Advice。說明如下：

* Beginning：程式啟動時執行，於 app().run() 完成後立即執行。此時所有 controller、filter、plugin、database client 均已建立，可於此取得物件引用或初始化。Advice 僅執行一次，型態為 `void()`，註冊介面為 `registerBeginningAdvice`。
* NewConnection：每個新 TCP 連線建立時執行，Advice 型態為 `bool(const trantor::InetAddress &, const trantor::InetAddress &)`，第一參數為遠端地址，第二為本地地址。回傳 false 則斷開連線。註冊介面為 `registerNewConnectionAdvice`。
* HttpResponseCreation：每個 HTTP Response 物件建立時執行，Advice 型態為 `void(const HttpResponsePtr &)`，參數為新建立物件。可統一操作所有 Response（如加 header），包含 404 或內部錯誤回應。註冊介面為 `registerHttpResponseCreationAdvice`。
* Sync：HTTP 請求處理最前端，Advice 型態為 `HttpRequestPtr(const HttpRequestPtr &)`，可回傳非空 Response 以攔截請求。註冊介面為 `registerSyncAdvice`。
* Pre-Routing：於框架尋找請求處理器前執行，Advice 型態有兩種：`void(const HttpRequestPtr &,AdviceCallback &&,AdviceChainCallback &&)`（同 Filter 的 doFilter，可攔截或放行請求）及 `void(const HttpRequestPtr &)`（無攔截能力、效能較高）。註冊介面為 `registerPreRoutingAdvice`。
* Post-Routing：找到處理器後、HTTP 方法檢查及 Filter 處理前執行，Advice 型態同上，註冊介面為 `registerPostRoutingAdvice`。
* Pre-Handling：通過所有 Filter 後、處理器執行前，Advice 型態同上，註冊介面為 `registerPreHandlingAdvice`。
* Post-Handling：處理器執行完產生 Response 後、送出前執行，Advice 型態為 `void(const HttpRequestPtr &, const HttpResponsePtr &)`，註冊介面為 `registerPostHandlingAdvice`。

### AOP 示意圖

下圖展示上述後四個插入點於 HTTP Request 處理流程中的位置，紅色圓點為插入點，綠色箭頭為非同步呼叫路徑。

![](images/AOP.png)

# 下一步：[效能測試](/CHN/CHN-14-性能测试)

[English](/ENG/ENG-01-Overview)

**Drogon** 是一個基於 C++17/20 的 HTTP 應用框架，使用 Drogon 可以方便地用 C++ 建構各類型的 Web 應用伺服器程式。

Drogon 主要支援的平台為 Linux，也支援 macOS、FreeBSD 及 Windows。

其主要特色如下：

* 網路層採用基於 epoll（macOS/FreeBSD 為 kqueue）的非阻塞 IO 架構，提供高併發、高效能的網路 IO。詳情請參考 [TFB 測試結果](https://www.techempower.com/benchmarks/#section=data-r19&hw=ph&test=composite)；
* 全非同步程式設計模式；
* 支援 HTTP/1.0、HTTP/1.1（伺服器端與客戶端）；
* 以 template 實作簡易的反射機制，讓主程式框架、控制器（controller）與視圖（view）完全解耦；
* 支援 cookies 與內建 session 管理；
* 支援後端渲染，將控制器產生的資料交給視圖產生 HTML 頁面，視圖由 CSP 樣板檔案描述，透過 CSP 標籤將 C++ 程式碼嵌入 HTML 頁面，並由 drogon 的指令工具在編譯階段自動產生 C++ 程式碼並編譯；
* 支援執行期的視圖頁面動態載入（動態編譯與載入 so 檔案）；
* 非常方便且彈性的路徑（path）到控制器處理函式（handler）映射方案；
* 支援過濾器（filter）鏈，方便在控制器前執行統一邏輯（如登入驗證、HTTP Method 約束驗證等）；
* 支援 HTTPS（基於 OpenSSL 實作）；
* 支援 WebSocket（伺服器端與客戶端）；
* 支援 JSON 格式請求與回應，對 RESTful API 開發非常友善；
* 支援檔案下載與上傳，支援 sendfile 系統呼叫；
* 支援 gzip/brotli 壓縮傳輸；
* 支援 pipelining；
* 提供輕量級的指令工具 drogon_ctl，協助簡化各類型專案建立與視圖程式碼產生流程；
* 基於非阻塞 IO 實作的非同步資料庫讀寫，目前支援 PostgreSQL 與 MySQL（MariaDB）資料庫；
* 基於執行緒池實作 sqlite3 資料庫的非同步讀寫，提供與上述資料庫相同的介面；
* 支援 ARM 架構；
* 方便的輕量級 ORM 實作，支援常規的物件到資料庫的雙向映射操作；
* 支援插件，可透過設定檔在載入期動態安裝與卸載；
* 支援內建插入點的 AOP（面向切面程式設計）

# 下一步：[安裝 drogon](/CHN/CHN-02-安裝)

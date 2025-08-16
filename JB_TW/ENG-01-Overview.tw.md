# 概述

[原文：ENG-01-Overview.md](/ENG/ENG-01-Overview.md)

**Drogon** 是一個基於 C++17/20 的 HTTP 應用程式框架。Drogon 可用於輕鬆建構各類型的 Web 應用伺服器程式，採用 C++ 開發。

**Drogon** 這個名字來自美國影集《權力遊戲》裡我很喜歡的一隻龍。

Drogon 主要運行於 Linux 平台，同時也支援 Mac OS、FreeBSD 及 Windows。

其主要特色如下：

* 採用基於 epoll（macOS/FreeBSD 下為 kqueue）的非阻塞 I/O 網路函式庫，提供高併發、高效能的網路 I/O，詳情請參考 [TFB 測試結果](https://www.techempower.com/benchmarks/#section=data-r19&hw=ph&test=composite)；
* 提供完整的非同步程式設計模式；
* 支援 Http1.0/1.1（伺服器端與客戶端）；
* 基於模板，實作簡易反射機制，完全解耦主程式框架、控制器與視圖；
* 支援 Cookie 與內建 Session；
* 支援後端渲染，控制器產生資料交由視圖產生 Html 頁面。視圖以 CSP 模板檔描述，可透過 CSP 標籤將 C++ 程式碼嵌入 Html 頁面，並由 drogon 命令列工具自動產生 C++ 程式碼檔案以供編譯；
* 支援視圖頁面動態載入（執行時動態編譯與載入）；
* 提供方便靈活的路由解決方案，從路徑對應至控制器處理函式；
* 支援過濾器鏈，便於在處理 HTTP 請求前執行統一邏輯（如登入驗證、HTTP 方法限制等）；
* 支援 https（基於 OpenSSL）；
* 支援 WebSocket（伺服器端與客戶端）；
* 支援 JSON 格式請求與回應，對 Restful API 應用開發非常友善；
* 支援檔案下載與上傳；
* 支援 gzip、brotli 壓縮傳輸；
* 支援管線化（pipelining）；
* 提供輕量級命令列工具 drogon_ctl，簡化 Drogon 各類型類別建立與視圖程式碼產生；
* 支援非阻塞 I/O 非同步讀寫資料庫（PostgreSQL 及 MySQL/MariaDB 資料庫）；
* 支援基於執行緒池的 sqlite3 資料庫非同步讀寫；
* 支援 ARM 架構；
* 提供方便輕量的 ORM 實作，支援物件與資料庫雙向映射；
* 支援插件，可於載入時透過設定檔安裝；
* 內建 AOP（面向切面程式設計）支援。

# 下一步: [安裝 drogon](ENG-02-Installation.tw.md)

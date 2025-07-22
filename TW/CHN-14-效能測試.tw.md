[English](/ENG/ENG-14-Benchmarks)

## 效能測試

作為 C++ 的 Http 應用框架，效能是重點之一，本節介紹 Drogon 的簡易測試與成績。

### 測試環境

* 系統：Linux CentOS 7.4
* 設備：Dell 伺服器，CPU 兩顆 Intel(R) Xeon(R) E5-2670 @ 2.60GHz，16 核 32 線程
* 記憶體：64GB
* gcc 版本：7.3.0

### 測試方案與結果

僅測試 drogon 框架效能，controller 處理盡量簡化，僅建立一個 HttpSimpleController，註冊於 `/benchmark` 路徑。controller 對所有請求回傳 `<p>Hello, world!</p>`，drogon 執行緒數設為 16。handler 程式如下，可於 `drogon/examples/benchmark` 目錄找到原始碼：

```c++
void BenchmarkCtrl::asyncHandleHttpRequest(const HttpRequestPtr &req, std::function<void (const HttpResponsePtr &)> &&callback)
{
    //write your application logic here
    auto resp = HttpResponse::newHttpResponse();
    resp->setBody("<p>Hello, world!</p>");
    resp->setExpiredTime(0);
    callback(resp);
}
```

對比測試選用 nginx，採 nginx+module 原始碼編譯，撰寫 hello_world_module，worker_processes 設為 16。

測試工具為高效能 HTTP 壓力測試工具 `httpress`。

調整 httpress 參數，每組測試五次，記錄每秒處理請求數最大值與最小值。結果如下表：

| 命令行                                       | 說明                                              | Drogon(千 QPS) | nginx(千 QPS) |
| :------------------------------------------- | :------------------------------------------------ | :------------: | :-----------: |
| httpress -c 100 -n 1000000 -t 16 -k -q URL   | 100 連線，100 萬請求，16 執行緒，Keep-Alive       |    561/552     |    330/329    |
| httpress -c 100 -n 1000000 -t 12 -q URL      | 100 連線，100 萬請求，12 執行緒，每請求新連線     |    140/135     |     31/49     |
| httpress -c 1000 -n 1000000 -t 16 -k -q URL  | 1000 連線，100 萬請求，16 執行緒，Keep-Alive      |    573/565     |    333/327    |
| httpress -c 1000 -n 1000000 -t 16 -q URL     | 1000 連線，100 萬請求，16 執行緒，每請求新連線    |    155/143     |     52/50     |
| httpress -c 10000 -n 4000000 -t 16 -k -q URL | 10000 連線，400 萬請求，16 執行緒，Keep-Alive     |    512/508     |    316/314    |
| httpress -c 10000 -n 1000000 -t 16 -q URL    | 10000 連線，100 萬請求，16 執行緒，每請求新連線   |    143/141     |     43/40     |

可見，客戶端使用 Keep-Alive 時，一連線可送多請求，drogon 每秒可處理 50 多萬次請求，成績相當優異。每請求新連線時，CPU 多耗於 TCP 建立與斷開，吞吐量降至每秒 14 萬次，屬正常。drogon 對比 nginx 成績明顯優勢，或許 nginx 配置未發揮最大效能，歡迎高手指正。

下圖為某次測試截圖：

![測試截圖](images/benchmark.png)

# 下一步：[Coz 分析](/CHN/CHN-15-Coz分析)

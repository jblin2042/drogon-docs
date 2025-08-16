## 認識 Drogon 的執行緒模型

[原文：ENG-FAQ-1-Understanding-drogon-threading-model.md](/ENG/ENG-FAQ-1-Understanding-drogon-threading-model.md)

Drogon 是高效能 C++ Web 應用框架，其效能部分來自於不隱藏底層執行緒模型，但也因此容易造成誤解，例如：為何某些阻塞呼叫後才送出回應、在同一事件迴圈呼叫阻塞網路函式導致死鎖等。本文說明造成這些現象的條件及如何避免。

### 事件迴圈與執行緒

Drogon 以執行緒池運作，每個執行緒有自己的事件迴圈（event loop），事件迴圈是 Drogon 核心。每個應用至少有兩個事件迴圈：主迴圈與工作迴圈。主迴圈永遠在主執行緒（啟動 main 的執行緒）上，負責啟動所有工作迴圈。以 hello world 範例：`app().run()` 啟動主迴圈並產生三個工作執行緒/迴圈。

```cpp
#include <drogon/drogon.h>
using namespace drogon;

int main()
{
    app().registerHandler("/", [](const HttpRequest& req
        , std::function<void (const HttpResponsePtr &)> &&callback) {
        auto resp = HttpResponse::newHttpResponse();
        resp->setBody("Hello wrold");
        callback(resp);
    });
    app().addListener("0.0.0.0", 80800);
    app().setNumThreads(3);
    app().run();
}
```

# 執行緒結構如下

```text
            .-------------.
            | app().run() | <main thread>
            :-------------:
                  |
            .-----v------.
            | MAIN LOOP  |
            :------------:
                  | Spawns
      .-----------+--------------.
      |           |              |
.-----v----.   .--v-------.  .---v----.
| Worker 1 |   | Worker 2 |  | etc... |
:----------:   :----------:  :--------:
 <thread 1>     <thread 2>   <thread ...>

```

工作迴圈數量取決於多項參數，如 HTTP server 執行緒數、非 fast DB/NoSQL 連線數等。每個事件迴圈本質上是任務佇列，具備：

- 從佇列讀取並執行任務（可由任意執行緒提交，完全 lock-free，不會競態）
- 處理網路事件
- 執行逾時定時器

閒置時則阻塞等待事件。

```cpp
// 在主迴圈排入兩個任務
trantor::EventLoop* loop = app().getLoop();
loop->queueInLoop([]{
    std::cout << "task1: I'm gonna wait for 5s\n";
    std::this_thread::sleep_for(5s);
    std::cout << "task1: hello!\n";
});
loop->queueInLoop([]{
    std::cout << "task2: world!\n";
});
#
```

執行結果：`task1: I'm gonna wait for 5s` 立即顯示，暫停 5 秒後 `task1: hello` 與 `task2: world` 一起顯示。

> 提示1：不要在事件迴圈執行阻塞 IO，其他任務會被延遲。

## 實務上的網路 IO

Drogon 幾乎所有物件都綁定事件迴圈（TCP 連線、HTTP client、DB client、快取等），為避免競態，所有 IO 都在對應事件迴圈執行。若由其他執行緒呼叫 IO，參數會被存下並以任務提交至正確事件迴圈。這意味著，例如在 HTTP handler 內呼叫遠端端點或 DB，client callback 不一定（通常不會）在 handler 執行緒。

```cpp
app().registerHandler("/send_req", [](const HttpRequest& req
    , std::function<void (const HttpResponsePtr &)> &&callback) {
    // 此 handler 在 HTTP server 執行緒

    // 建立在主迴圈運作的 HTTP client
    auto client = HttpClient::newHttpClient("https://drogon.org", app().getLoop());
    auto request = HttpRequest::newHttpRequest();
    client->sendRequest(request, [](ReqResult result, const HttpResponse& resp) {
        // 此 callback 在主執行緒
    });
});
```

因此若不注意，可能堵塞事件迴圈，例如在主迴圈建立大量 HTTP client 並發送請求，或在 DB callback 執行重運算，導致其他 DB 查詢延遲。

同理，HTTP server 也是如此。若回應由其他執行緒產生（如 DB callback），則回應會排入對應執行緒送出，而非立即送出。

> 提示2：注意計算放置位置，否則影響 throughput。

### 事件迴圈死鎖

理解 Drogon 設計後，死鎖很容易發生：在同一迴圈提交遠端 IO 並等待 callback（即 sync API）。它提交 IO 並等待 callback。

```cpp
app().registerHandler("/dead_lock", [](const HttpRequest& req
    , std::function<void (const HttpResponsePtr &)> &&callback) {
    auto currentLoop = app().getIOLoops()[app().getCurrentThreadIndex()];
    auto client = HttpClient::newHttpClient("https://drogon.org", currentLoop);
    auto request = HttpRequest::newHttpRequest();
    auto resp = client->sendRequest(resp); // 死鎖！呼叫同步介面
});
```

因此，如果你不清楚自己的程式碼實際上在做什麼，就可能會阻塞事件迴圈 (event loop)。比方說，在主迴圈 (main loop) 中建立大量的 HTTP 客戶端並發送所有外發請求；或是在資料庫的回呼函式 (callback) 中執行運算密集的函式，進而導致其他正在處理的資料庫查詢被阻擋。

示意圖：

```txt
 Worker 1       Main Loop        Worker2
.---------.    .----------.     .---------.
|         |    |          |     |         |
| req 1-. |    |----------|  .--+--req 2  |
|       :-+----+->        |  |  |         |
|         |    | send http|  |  |---------|
|---------| a.-|   req 1  |  |  |         |
|other req| s| |----------|  |  |         |
|---------| y| |      <---+--:  |         |
|         | n| |send http |     |         |
|         | c| |  req 2   |-.   |         |
|         |  | |----------| |a  |---------|
|         |  | |http resp1| |s  |other req|
|         |  :-|>compute  | |y  |---------|
|         |    |          | |n  |         |
|         | .--+-generate | |c  |         |
|         | |  | response | |   |         |
|         | |  |----------| |   |         |
|         | |  |http resp2|<:   |         |
|---------| |  | compute  |     |---------|
|response<|-:  |          |-----|>        |
|send back|    | generate |     |send resp|
|         |    | response |     | back    |
:---------:    :----------:     :---------:
```

DB、NoSQL callback 也一樣。幸好非 fast DB client 各自有執行緒，故可在 HTTP handler 安全執行同步查詢，但不可在同 client callback 執行同步查詢，否則也會死鎖。

> 提示3：同步 API 影響效能與安全，盡量避免。若必須用，請確保 client 在不同執行緒。

#### Fast DB client

Drogon 以效能為優先，fast DB client 共用 HTTP server 執行緒，省去跨執行緒切換提升效能，但不可執行同步查詢，否則死鎖。

### 協程救援

Drogon 開發常遇到：非同步 API 高效但難用，同步 API 易用但效能差且易出問題。Lambda 宣告冗長、語法不直覺，且程式充滿 callback。同步 API 雖簡潔但效能差。

```cpp
// drogon 非同步 DB API
auto db = app().getDbClient();
db->execSqlAsync("INSERT......", [db, callback](auto result){
    db->execSqlAsync("UPDATE .......", [callback](auto result){
        // 成功處理
    },
    [callback](const DbException& e) {
        // 失敗處理
    })
},
[callback](const DbException& e){
    // 失敗處理
})
```

對比：

```cpp
// drogon 同步 API，例外自動處理
db->execSqlSync("INSERT.....");
db->execSqlSync("UPDATE.....");
```

有沒有兩全其美的方法？C++20 協程就是答案。協程本質上是編譯器支援的 callback 包裝，讓程式看似同步，實則全程非同步。範例：

```cpp
co_await db->execSqlCoro("INSERT.....");
co_await db->execSqlCoro("UPDATE.....");
```

語法與同步 API 幾乎一樣，但效能更佳。建議優先使用協程（GCC >= 11，MSVC >= 16.25）。協程雖非萬靈丹，無法自動避免事件迴圈堵塞與競態，但更易除錯與理解非同步程式。

> 提示4：能用協程就用協程

### 小結

- 優先用 C++20 協程與 Fast DB 連線
- 同步 API 可能拖慢甚至死鎖事件迴圈
- 若必須用同步 API，請確保 client 在不同執行緒

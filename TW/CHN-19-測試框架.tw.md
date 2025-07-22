[English](/ENG/ENG-19-Testing-Framework)

# 測試框架

DrogonTest 是 Drogon 內建的極簡測試框架，支援基本的非同步與同步測試。它用於 Drogon 本身的單元測試與整合測試，也可用於測試基於 Drogon 開發的應用程式。DrogonTest 的語法受 [GTest](https://github.com/google/googletest) 與 [Catch2](https://github.com/catchorg/Catch2) 啟發。

你不一定要用 DrogonTest，也可選擇其他測試框架，但這是其中一個選項。

### 基本測試

先看一個簡單範例。有個函式計算某數以前所有自然數的總和，想測試其正確性。

```c++
// 告訴 DrogonTest 產生 `test::run()`，只在主檔案定義
#define DROGON_TEST_MAIN
#include <drogon/drogon_test.h>

int sum_all(int n)
{
    int result = 1;
    for(int i=2;i<n;i++) result += i;
    return result;
}

DROGON_TEST(Sum)
{
    CHECK(sum_all(1) == 1);
    CHECK(sum_all(2) == 3);
    CHECK(sum_all(3) == 6);
}

int main(int argc, char** argv)
{
    return drogon::test::run(argc, argv);
}
```

編譯並執行...看起來通過了，但其實有明顯錯誤。`sum_all(0)` 應為 0。可將此測試加入：

```c++
DROGON_TEST(Sum)
{
    CHECK(sum_all(0) == 0);
    CHECK(sum_all(1) == 1);
    CHECK(sum_all(2) == 3);
    CHECK(sum_all(3) == 6);
}
```

此時測試失敗：

```
In test case Sum
↳ /path/to/your/test/main.cc:47  FAILED:
  CHECK(sum_all(0) == 0)
With expansion
  1 == 0
```

框架會顯示表達式兩側的實際值，方便立即看出問題。修正如下：

```c++
int sum_all(int n)
{
    int result = 0;
    for(int i=1;i<n;i++) result += i;
    return result;
}
```

### 測試型態

DrogonTest 提供多種測試型態與操作。基本的 `CHECK()` 檢查表達式是否為真，否則輸出至主控台。`CHECK_THROWS()` 檢查是否丟出例外。`REQUIRE()` 則在不成立時直接 return。

| 失敗後/表達式 | 為真       | 丟出例外          | 未丟出例外      | 丟出特定例外         |
| ------------- | ---------- | ----------------- | ------------------ | -------------------- |
| 不做任何事    | CHECK      | CHECK_THROWS      | CHECK_NOTHROW      | CHECK_THROWS_AS      |
| return        | REQUIRE    | REQUIRE_THROWS    | REQUIRE_NOTHROW    | REQUIRE_THROWS_AS    |
| 協程 return   | CO_REQUIRE | CO_REQUIRE_THROWS | CO_REQUIRE_NOTHROW | CO_REQUIRE_THROWS_AS |
| 終止程序      | MANDATE    | MANDATE_THROWS    | MANDATE_NOTHROW    | MANDATE_THROWS_AS    |

例如測試檔案內容，若無法開啟檔案就不需繼續檢查，可用 REQUIRE 簡化：

```c++
DROGON_TEST(TestContent)
{
    std::ifstream in("data.txt");
    REQUIRE(in.is_open());
    // 取代
    // CHECK(in.is_open() == true);
    // if(in.is_open() == false)
    //    return;

    ...
}
```

`CO_REQUIRE` 用於協程。若操作會改變且無法復原全域狀態，可用 `MANDATE`，唯一合理做法是終止測試。

### 非同步測試

Drogon 是非同步網頁框架，因此 DrogonTest 也支援非同步測試。DrogonTest 透過 `TEST_CTX` 追蹤測試上下文，只需以**值**捕獲即可。例如測試遠端 API 是否成功回傳 JSON：

```c++
DROGON_TEST(RemoteAPITest)
{
    auto client = HttpClient::newHttpClient("http://localhost:8848");
    auto req = HttpRequest::newHttpRequest();
    req->setPath("/");
    client->sendRequest(req, [TEST_CTX](ReqResult res, const HttpResponsePtr& resp) {
        // 若請求未到達伺服器或伺服器回傳錯誤
        REQUIRE(res == ReqResult::Ok);
        REQUIRE(resp != nullptr);

        CHECK(resp->getStatusCode() == k200OK);
        CHECK(resp->contentType() == CT_APPLICATION_JSON);
    });
}
```

因測試框架支援 C++14/17，相容協程需包裝於 `AsyncTask` 或用 `sync_wait` 呼叫。

```c++
DROGON_TEST(RemoteAPITestCoro)
{
    auto api_test = [TEST_CTX]() {
        auto client = HttpClient::newHttpClient("http://localhost:8848");
        auto req = HttpRequest::newHttpRequest();
        req->setPath("/");

        auto resp = co_await client->sendRequestCoro(req);
        CO_REQUIRE(resp != nullptr);
        CHECK(resp->getStatusCode() == k200OK);
        CHECK(resp->contentType() == CT_APPLICATION_JSON);
    };

    sync_wait(api_test());
}
```

### 啟動 Drogon 事件迴圈

部分測試需 Drogon 事件迴圈運作，例如 HTTP 客戶端預設運作於全域事件迴圈。以下範例可處理多種情境，確保事件迴圈於測試開始前啟動：

```c++
int main()
{
    std::promise<void> p1;
    std::future<void> f1 = p1.get_future();

    // 於另一執行緒啟動主迴圈
    std::thread thr([&]() {
        // 事件迴圈啟動後 fulfill promise
        app().getLoop()->queueInLoop([&p1]() { p1.set_value(); });
        app().run();
    });

    // 事件迴圈啟動後才繼續
    f1.get();
    int status = test::run(argc, argv);

    // 要求事件迴圈結束並等待
    app().getLoop()->queueInLoop([]() { app().quit(); });
    thr.join();
    return status;
}
```

### CMake 整合

如同多數測試框架，DrogonTest 可整合至 CMake。`ParseAndAddDrogonTests` 會將源碼中的測試加入 CMake 的 CTest 框架。

```cmake
include(ParseAndAddDrogonTest) # 也會載入 ParseAndAddDrogonTests
add_executable(mytest main.cpp)
target_link_libraries(mytest PRIVATE Drogon::Drogon)
ParseAndAddDrogonTests(mytest)
```

現在可透過建置系統（如 Makefile）執行測試：

```bash
❯ make test
Running tests...
Test project path/to/your/test/build/
      Start  1: Sum
 1/1  Test  #1: Sum ....................................   Passed    0.00 sec
```

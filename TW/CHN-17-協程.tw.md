[English](/ENG/ENG-17-Coroutines)

# 協程

Drogon 自 1.4 版起支援 [C++ coroutines][1]（協程）。它提供扁平化非同步執行流程的方法，可避免著名的 callback hell。透過協程，非同步程式設計如同同步般簡單（同時保有高效能）。

## 術語

本文不解釋協程原理，只介紹如何於 Drogon 使用。協程常見術語如下：

**協程（Coroutine）**：可暫停並於之後恢復執行的函式。  
**Return**：普通函式結束並回傳值；協程需回傳含 `promise_type` 的物件（本文稱 _resumable_ 型別），用於恢復執行。  
**(co_)yield**：協程暫停並回傳值。  
**co_return**：協程結束並回傳值（如有）。  
**(co_)await**：協程等待結果，若結果未就緒（如需網路請求），則暫停執行，執行緒處理其他任務。結果就緒時恢復協程（不一定於同一執行緒）。

## 啟用協程

Drogon 協程功能為 header-only，即使編譯 drogon 庫的編譯器不支援協程，使用者仍可用協程。啟用方式依編譯器而異，GCC >=10.0 可用 `-std=c++20 -fcoroutines`。MSVC（測試於 19.25）需設 `/std:c++latest` 且不可設 `/await`。例如 GCC 可用：

```shell
cmake .. -DCMAKE_CXX_FLAGS="-fcoroutines"
```

注意截至 clang 12.0，Drogon 協程尚不支援 clang。GCC 11 啟用 c++20 標準時預設支援協程，編譯 drogon 應用無需特別設定。GCC 10 雖可編譯執行協程，但有 bug 會導致協程 frame 未釋放、記憶體洩漏。

## 使用協程

協程效能與邏輯等同非同步介面，但介面形式為同步。Drogon 中每個協程函式（或可於協程中 co_await 的函式）介面皆為同步介面加上 `Coro` 後綴。例如 `db->execSqlSync()` 對應 `db->execSqlCoro()`，`client->sendRequest()` 對應 `client->sendRequestCoro()`。上述函式皆回傳 _awaitable_ 物件，`co_await` 可立即或稍後恢復協程並取得結果，等待期間執行緒可處理其他 IO 等任務。程式看似同步，實則非同步。

例如查詢資料庫用戶數：

```c++
app.registerHandler("/num_users",
    [](HttpRequestPtr req, std::function<void(const HttpResponsePtr&)> callback) -> Task<>
    //                                回傳值必須是可恢復 (resumable) 的型別 (框架已封裝好) ^^^
{
    auto sql = app().getDbClient();
    try
    {
        auto result = co_await sql->execSqlCoro("SELECT COUNT(*) FROM users;");
        size_t num_users = result[0][0].as<size_t>();
        auto resp = HttpResponse::newHttpResponse();
        resp->setBody(std::to_string(num_users));
        callback(resp);
    }
    catch(const DrogonDbException &err)
    {
        // 例外處理也可以像同步介面那樣正常運作
        auto resp = HttpResponse::newHttpResponse();
        resp->setBody(err.base().what());
        callback(resp);
    }
    co_return; // 此陳述不是必要的，因為它位於協程的結尾處。由於回傳值型別為 Task<void>，這裡不需要回傳任何值。
}

```

注意事項：

1. 使用 co_await 的 handler 必為協程，回傳型別不可為 void，須用框架封裝的 Task<T> 模板。
2. 普通函式 return 於協程中須改為 co_return。
3. 協程參數須用值傳遞，不可用引用。

Task<T> 模板遵循 C++ coroutine 標準，使用者只需知若協程產生 T 型結果，回傳型別即為 `Task<T>`。

值傳遞為協程非同步執行的約束，編譯器會自動複製（或 move）參數至協程 frame，確保恢復時可用。引用參數僅複製地址，除非確知參數生命週期覆蓋整個協程，否則請用值型別。

若希望直接回傳 response 而非用 callback，可用 co_return，Drogon 支援此寫法，但效能最多損失約 8%（相較 callback），請依需求選用。上述例子可改寫如下：

```c++
app.registerHandler("/num_users",
    [](const HttpRequestPtr& req) -> Task<HttpResponsePtr>
    //               這裡直接回傳 response 物件 ^^^
{
    auto sql = app().getDbClient();
    try
    {
        auto result = co_await sql->execSqlCoro("SELECT COUNT(*) FROM users;");
        size_t num_users = result[0][0].as<size_t>();
        auto resp = HttpResponse::newHttpResponse();
        resp->setBody(std::to_string(num_users));
        co_return resp;
    }
    catch(const DrogonDbException &err)
    {
        // 例外處理也可以像同步介面那樣正常運作
        auto resp = HttpResponse::newHttpResponse();
        resp->setBody(err.base().what());
        co_return resp;
    }
});

```

目前 websocket 控制器尚不支援協程，若有需求請至 github 提 issue。

## 常見缺陷

協程常見陷阱：

- ### 於函式中使用 lambda 捕獲協程

  Lambda 捕獲與協程生命週期不同。協程 frame 會持續至被銷毀，匿名 lambda 通常於呼叫後即銷毀。協程非同步特性導致協程生命週期可能遠長於 lambda。例如下例，lambda 於等待 SQL 完成後即銷毀，協程 frame 則持續等待 SQL，SQL 完成時 lambda 捕獲已失效。

  ```c++
    app().getLoop()->queueInLoop([num] () -> AsyncTask {
        auto db = app().getDbClient();
        co_await db->execSqlCoro("DELETE FROM customers WHERE last_login < CURRENT_TIMESTAMP - INTERVAL $1 DAY", std::to_string(num));
        // Lambda 物件及其捕獲的內容在協程暫停時就會被銷毀。
        LOG_INFO << "Remove old customers that have no activity for more than " << num << "days"; // use-after-free (存取已被釋放的記憶體)
    });
    // 錯誤的寫法，這會導致程式崩潰
  ```

  Drogon 提供 `async_func` 包裝 lambda，確保生命週期：

  ```c++
    app().getLoop()->queueInLoop(async_func([num]() -> Task<void> {
    //                             ^^^^^^^^^^^^^^^^^^^^^^^^^ 用 async_func 包裝並回傳一個 Task<>
        auto db = app().getDbClient();
        co_await db->execSqlCoro("DELETE FROM customers WHERE last_login < CURRENT_TIMESTAMP - INTERVAL $1 DAY", std::to_string(num));
        LOG_INFO << "Remove old customers that have no activity for more than " << num << "days";
    }));
    // 正確的寫法
  ```

- ### 於函式中以引用傳遞／捕獲至協程

  C++ 中以引用傳遞可減少不必要複製，但由函式傳至協程常致問題，因協程為非同步且生命週期較長。例如下例：

  ```cpp
    void removeCustomers(const std::string& customer_id)
    {
        async_run([&customer_id]() -> Task<void> {
            //      ^^^^ 不要透過參考 (by reference) 傳遞/捕獲物件到協程中
            // 除非您能確保該物件的生命週期比協程更長

            auto db = app().getDbClient();
            co_await db->execSqlCoro("DELETE FROM customers WHERE customer_id = $1", customer_id);
            // `customer_id` 在等待 SQL 時就已離開作用域。程式會在此處崩潰
            co_await db->execSqlCoro("DELETE FROM orders WHERE customer_id = $1", customer_id);
        });
    }
  ```

  但由協程傳遞引用則可：

  ```cpp
  Task<> removeCustomers(const std::string& customer_id)
    {
        auto db = app().getDbClient();
        co_await db->execSqlCoro("DELETE FROM customers WHERE customer_id = $1", customer_id);
        co_await db->execSqlCoro("DELETE FROM orders WHERE customer_id = $1", customer_id);
    }

    Task<> findUnwantedCustomers()
    {
        auto db = app().getDbClient();
        auto list = co_await db->execSqlCoro("SELECT customer_id from customers "
            "WHERE customer_score < 5;");
        for(const auto& customer : list)
            co_await removeCustomers(customer["customer_id"].as<std::string>());
            //                               ^^^^^^^^^^^^^^^^^
            // 雖然這是一個 const reference，但因為我們是從一個協程中呼叫它，
            // 所以這種寫法完全沒問題，而且是建議的作法。
    }
  ```

[1]: https://en.cppreference.com/w/cpp/language/coroutines

# 下一步：[Redis](/CHN/CHN-18-Redis)

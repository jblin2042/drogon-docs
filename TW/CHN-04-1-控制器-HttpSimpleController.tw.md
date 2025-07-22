[English](/ENG/ENG-04-1-Controller-HttpSimpleController)

# 可由 drogon_ctl 產生

可以用 `drogon_ctl` 指令工具快速產生基於 `HttpSimpleController` 的自訂類別原始檔，指令格式如下：

```bash
drogon_ctl create controller <[namespace::]class_name>
```

我們建立一個名為 `TestCtrl` 的控制器：

```bash
drogon_ctl create controller TestCtrl
```

會新增兩個檔案：TestCtrl.h 和 TestCtrl.cc，以下說明這兩個檔案內容。

TestCtrl.h 範例如下：

```c++
#pragma once
#include <drogon/HttpSimpleController.h>
using namespace drogon;
class TestCtrl:public drogon::HttpSimpleController<TestCtrl>
{
public:
    virtual void asyncHandleHttpRequest(const HttpRequestPtr &req,
                                        std::function<void (const HttpResponsePtr &)> &&callback)override;
    PATH_LIST_BEGIN
    // 在此定義路徑
    //PATH_ADD("/path","filter1","filter2",HttpMethod1,HttpMethod2...);
    PATH_LIST_END
};
```

TestCtrl.cc 範例如下：

```c++
#include "TestCtrl.h"
void TestCtrl::asyncHandleHttpRequest(const HttpRequestPtr &req,
                                      std::function<void (const HttpResponsePtr &)> &&callback)
{
    //write your application logic here
    // 在此撰寫應用邏輯
}
```

每個 HttpSimpleController 類別只能定義一個 HTTP 請求處理函式（handler），而且必須用虛擬函式重載。

從 URL 路徑到處理函式的路由（或稱映射）由巨集完成，可用 `PATH_ADD` 巨集新增多重路徑映射，所有 `PATH_ADD` 語句需夾在 `PATH_LIST_BEGIN` 和 `PATH_LIST_END` 巨集之間。

第一個參數是映射的路徑，路徑後面的參數是對該路徑的約束，目前支援兩種約束：一種是 `HttpMethod` 類型，表示該路徑允許的 HTTP 方法，可設定零個或多個；另一種是 `HttpFilter` 類別名稱，該物件執行特定過濾操作，也可設定零個或多個，兩種類型順序不限，框架會自動處理型別匹配。關於 Filter，請參考 [中介軟體與過濾器](/CHN/CHN-05-中间件和过滤器)。

使用者可以將同一個 Simple Controller 註冊到多個路徑，也可在同一路徑上註冊多個 Simple Controller（透過 HTTP method 區分）。

你可以定義一個 HttpResponse 物件，然後用 callback() 回傳即可：

```c++
    // 在此撰寫應用邏輯
    //write your application logic here
    auto resp=HttpResponse::newHttpResponse();
    resp->setStatusCode(k200OK);
    resp->setContentTypeCode(CT_TEXT_HTML);
    resp->setBody("Your Page Contents");
    callback(resp);
```

> **上述路徑到處理函式的映射是在編譯期完成的，事實上 drogon 框架也提供執行期完成映射的介面，執行期映射可讓使用者透過設定檔或其他介面完成或修改映射關係而無需重新編譯（基於效能考量，禁止在執行 app().run() 之後再註冊任何映射）。**

# 下一步：[HttpController](/CHN/CHN-04-2-控制器-HttpController)

[English](/ENG/ENG-07-Session)

# 會話（Session）

`會話（Session）` 是 Web 應用的重要概念，用於在伺服器端保存用戶端狀態，通常與瀏覽器的 `cookie` 搭配使用。drogon 提供會話支援，預設**關閉**會話功能，你也可以用以下介面開啟或關閉：

```c++
void disableSession();
void enableSession(const size_t timeout=0, Cookie::SameSite sameSite=Cookie::SameSite::kNull);
```

皆透過 `HttpAppFramework` 單例呼叫，timeout 參數代表會話失效時間（秒），預設 1200 秒，即用戶 20 分鐘未訪問則會話失效。timeout 設為 0 表示 drogon 會在整個生命週期保留用戶會話。

開啟會話前請確認用戶端支援 cookie，否則 drogon 會為每次不含 SessionID 的請求建立新會話，浪費記憶體與運算資源。

### 會話物件

drogon 的會話物件型別為 `drogon::Session`，與 `HttpViewData` 類似，可用關鍵字存取任意型別物件，支援並行讀寫。詳細用法請參考 Session 類說明。

drogon 框架會將會話物件放入 `HttpRequest` 物件中傳給使用者，可用以下介面取得 Session 物件：

```c++
SessionPtr session() const;
```

取得的是 Session 智慧指標，可用來存取各種物件。

### 會話範例

以下範例實作需會話支援的功能，例如限制用戶存取頻率：某次存取後，10 秒內再次存取則回傳錯誤，否則回傳 ok。需在會話中記錄上次存取時間，與本次存取時間比較即可。

建立一個 Filter 實作此功能，假設類名為 TimeFilter，實作如下：

```c++
#include "TimeFilter.h"
#include <trantor/utils/Date.h>
#include <trantor/utils/Logger.h>
#define VDate "visitDate"
void TimeFilter::doFilter(const HttpRequestPtr &req,
                          FilterCallback &&cb,
                          FilterChainCallback &&ccb)
{
    trantor::Date now=trantor::Date::date();
    LOG_TRACE<<"";
    if(req->session()->find(VDate))
    {
        auto lastDate=req->session()->get<trantor::Date>(VDate);
        LOG_TRACE<<"last:"<<lastDate.toFormattedString(false);
        req->session()->modify<trantor::Date>(VDate,
                                        [now](trantor::Date &vdate) {
                                            vdate = now;
                                        });
        LOG_TRACE<<"update visitDate";
        if(now>lastDate.after(10))
        {
            // 10 秒後可再次存取
            ccb();
            return;
        }
        else
        {
            Json::Value json;
            json["result"]="error";
            json["message"]="Access interval should be at least 10 seconds";
            auto res=HttpResponse::newHttpJsonResponse(json);
            cb(res);
            return;
        }
    }
    LOG_TRACE<<"first access,insert visitDate";
    req->session()->insert(VDate,now);
    ccb();
}
```

再註冊一個 lambda 到 `/slow` 路徑，並加上 TimeFilter，程式如下：

```c++
drogon::HttpAppFramework::instance()
    .registerHandler
     ("/slow",
      [=](const HttpRequestPtr &req,
          std::function<void (const HttpResponsePtr &)> &&callback)
          {
              Json::Value json;
              json["result"]="ok";
              auto resp=HttpResponse::newHttpJsonResponse(json);
              callback(resp);
          },
          {Get,"TimeFilter"}
      );
```

呼叫框架介面開啟會話：

```c++
drogon::HttpAppFramework::instance().enableSession(1200);
```

用 cmake 重新編譯整個專案，執行 webapp，即可在瀏覽器測試效果。

# 下一步：[資料庫](/CHN/CHN-08-0-数据库-概述)

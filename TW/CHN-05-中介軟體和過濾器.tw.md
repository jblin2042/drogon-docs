[English](/ENG/ENG-05-Middleware-and-Filter)

# 中介軟體

中介軟體（middleware）和過濾器（filter）可以協助開發者提升程式設計效率。在 HttpController 的[範例](/CHN/CHN-04-2-控制器-HttpController)中，getInfo 方法在回傳使用者資訊前應先驗證使用者是否已登入。雖然可以直接在 getInfo 方法內寫驗證邏輯，但這屬於通用邏輯，許多 API 都會用到，應該獨立出來並在 handler 執行前配置，這就是 filter 的用途。

drogon 的中介軟體採用洋蔥圈模型，框架完成 URL 路徑比對後，會依序呼叫註冊在該路徑上的中介軟體。每個中介軟體可選擇攔截或放行請求，並加入前置、後置處理邏輯。
若有中介軟體攔截請求，該請求不會繼續進入洋蔥圈內層，對應的 handler 也不會被呼叫，但仍會執行外層中介軟體的後置處理邏輯。

過濾器其實是省略後置操作的中介軟體，經過包裝後提供給使用者。中介軟體和過濾器可在註冊路徑時混合使用。

### 內建中介軟體／過濾器

drogon 內建常用過濾器如下：

* `drogon::IntranetIpFilter`：只放行內網 IP 發送的 HTTP 請求，否則回傳 404 頁面；
* `drogon::LocalHostFilter`：只放行本機 127.0.0.1 或 ::1 發送的 HTTP 請求，否則回傳 404 頁面；

### 自訂中介軟體／過濾器

* #### 中介軟體定義

  使用者可自訂中介軟體，需繼承 `HttpMiddleware` 類模板，模板型別即子類型。例如要為某些路由開啟跨域支援，可定義如下：

  ```c++
  class MyMiddleware : public HttpMiddleware<MyMiddleware>
  {
  public:
      MyMiddleware(){};  // 不可省略建構子

      void invoke(const HttpRequestPtr &req,
                  MiddlewareNextCallback &&nextCb,
                  MiddlewareCallback &&mcb) override
      {
          const std::string &origin = req->getHeader("origin");
          if (origin.find("www.some-evil-place.com") != std::string::npos)
          {
              // 直接攔截
              mcb(HttpResponse::newNotFoundResponse(req));
              return;
          }
          // 執行進入下一層前的邏輯
          nextCb([mcb = std::move(mcb)](const HttpResponsePtr &resp) {
              // 執行下一層回傳後的邏輯
              resp->addHeader("Access-Control-Allow-Origin", origin);
              resp->addHeader("Access-Control-Allow-Credentials","true");
              mcb(resp);
          });
      }
  };
  ```

  需重載父類的 `invoke` 虛擬函式以實作中介軟體邏輯。

  此虛擬函式有三個參數：

  * **req**：HTTP 請求；
  * **nextCb**：進入內層的回呼函式，呼叫代表繼續進入洋蔥圈內層，執行下個中介軟體或最終 handler。
    呼叫 nextCb 時需傳入另一函式，當從內層回傳時會被呼叫，並傳入內層回傳的 HttpResponsePtr。
  * **mcb**：回到外層的回呼函式，呼叫代表返回洋蔥圈外層。若只呼叫 mcb 而不呼叫 nextCb，代表攔截請求，直接回到外層。

* #### 過濾器定義

  使用者可自訂過濾器，需繼承 HttpFilter 類模板，模板型別即子類型。例如要做登入驗證 LoginFilter，可定義如下：

  ```c++
  class LoginFilter:public drogon::HttpFilter<LoginFilter>
  {
  public:
      void doFilter(const HttpRequestPtr &req,
                    FilterCallback &&fcb,
                    FilterChainCallback &&fccb) override ;
  };
  ```

  也可用 `drogon_ctl` 指令建立過濾器，詳見 [drogon_ctl](/CHN/CHN-11-drogon_ctl命令#过滤器创建)。

  需重載父類的 doFilter 虛擬函式以實作過濾器邏輯。

  此虛擬函式有三個參數：

  * **req**：HTTP 請求；
  * **fcb**：過濾器回呼函式，型別為 void (HttpResponsePtr)，當判定請求不合法時，透過此回呼回傳特定回應給瀏覽器；
  * **fccb**：過濾器鏈回呼函式，型別為 void()，當判定請求合法時，透過此回呼通知 drogon 執行下個過濾器或最終 handler；

  具體實作可參考 drogon 內建過濾器。

* #### 中介軟體／過濾器註冊

  中介軟體／過濾器總是隨 controller 註冊進行，前述註冊 handler 的巨集（PATH_ADD、METHOD_ADD 等）都可在最後加上一個或多個中介軟體／過濾器名稱。例如將前述 getInfo 方法註冊行改為：

  ```c++
  METHOD_ADD(User::getInfo,"/{1}/info?token={2}",Get,"LoginFilter","MyMiddleware");
  ```

  則路徑比對成功後，需同時滿足以下條件 getInfo 方法才會被呼叫：

  1. 請求必須是 HTTP GET；
  2. 請求方必須已登入；

  可見中介軟體／過濾器的配置與註冊非常簡單，註冊 controller 時不需引用中介軟體標頭檔，中介軟體與 controller 也完全解耦。

  > **注意：若中介軟體／過濾器定義在命名空間內，註冊時必須寫全命名空間**

# 下一步：[視圖](/CHN/CHN-06-視圖)

[English](/ENG/ENG-04-2-Controller-HttpController)

# 產生

可以用 `drogon_ctl` 指令工具快速產生基於 `HttpController` 的自訂類別原始檔，指令格式如下：

```bash
drogon_ctl create controller -h <[namespace::]class_name>
```

我們建立一個在 `demo v1` 命名空間下、名稱為 `User` 的控制器：

```bash
drogon_ctl create controller -h demo::v1::User
```

會新增兩個檔案：demo_v1_User.h 和 demo_v1_User.cc

demo_v1_User.h 範例如下：

```c++
#pragma once

#include <drogon/HttpController.h>

using namespace drogon;

namespace demo
{
namespace v1
{
class User : public drogon::HttpController<User>
{
  public:
    METHOD_LIST_BEGIN
    // 用 METHOD_ADD 新增自訂處理函式
    // METHOD_ADD(User::get, "/{2}/{1}", Get); // 路徑為 /demo/v1/User/{arg2}/{arg1}
    // METHOD_ADD(User::your_method_name, "/{1}/{2}/list", Get); // 路徑為 /demo/v1/User/{arg1}/{arg2}/list
    // ADD_METHOD_TO(User::your_method_name, "/absolute/path/{1}/{2}/list", Get); // 路徑為 /absolute/path/{arg1}/{arg2}/list

    METHOD_LIST_END
    // 處理函式宣告範例：
    // void get(const HttpRequestPtr& req, std::function<void (const HttpResponsePtr &)> &&callback, int p1, std::string p2);
    // void your_method_name(const HttpRequestPtr& req, std::function<void (const HttpResponsePtr &)> &&callback, double p1, int p2) const;
};
}
}
```

demo_v1_User.cc 範例如下：

```c++
#include "demo_v1_User.h"

using namespace demo::v1;

// 在此新增處理函式定義
```

### 使用方式

我們編輯這兩個檔案，說明如下。

demo_v1_User.h 範例如下：

```c++
#pragma once

#include <drogon/HttpController.h>

using namespace drogon;

namespace demo
{
namespace v1
{
class User : public drogon::HttpController<User>
{
  public:
    METHOD_LIST_BEGIN
    // 用 METHOD_ADD 新增自訂處理函式
    METHOD_ADD(User::login,"/token?userId={1}&passwd={2}",Post);
    METHOD_ADD(User::getInfo,"/{1}/info?token={2}",Get);
    METHOD_LIST_END
    // 處理函式宣告範例：
    void login(const HttpRequestPtr &req,
               std::function<void (const HttpResponsePtr &)> &&callback,
               std::string &&userId,
               const std::string &password);
    void getInfo(const HttpRequestPtr &req,
                 std::function<void (const HttpResponsePtr &)> &&callback,
                 std::string userId,
                 const std::string &token) const;
};
}
}
```

demo_v1_User.cc 範例如下：

```c++
#include "demo_v1_User.h"

using namespace demo::v1;

// 在此新增處理函式定義

void User::login(const HttpRequestPtr &req,
           std::function<void (const HttpResponsePtr &)> &&callback,
           std::string &&userId,
           const std::string &password)
{
    LOG_DEBUG<<"User "<<userId<<" login";
    // 認證演算法、讀取資料庫、驗證身分等...
    //...
    Json::Value ret;
    ret["result"]="ok";
    ret["token"]=drogon::utils::getUuid();
    auto resp=HttpResponse::newHttpJsonResponse(ret);
    callback(resp);
}
void User::getInfo(const HttpRequestPtr &req,
             std::function<void (const HttpResponsePtr &)> &&callback,
             std::string userId,
            const std::string &token) const
{
    LOG_DEBUG<<"User "<<userId<<" get his information";
    // 驗證 token 有效性等
    // 讀取資料庫或快取取得使用者資訊
    Json::Value ret;
    ret["result"]="ok";
    ret["user_name"]="Jack";
    ret["user_id"]=userId;
    ret["gender"]=1;
    auto resp=HttpResponse::newHttpJsonResponse(ret);
    callback(resp);
}
```

每個 `HttpController` 類別可定義多個 HTTP 請求處理函式（handler），因為函式數量不限，無法用虛擬函式重載，需將處理函式本身（而非類別）註冊到框架。

* #### 路徑映射

  URL 路徑到處理函式的映射由巨集完成，可用 `METHOD_ADD` 或 `ADD_METHOD_TO` 巨集新增多重路徑映射，所有 `METHOD_ADD` 和 `ADD_METHOD_TO` 語句需夾在 `METHOD_LIST_BEGIN` 和 `METHOD_LIST_END` 巨集之間。

  `METHOD_ADD` 巨集會自動將命名空間和類名作為路徑前綴，所以本例中 login 函式註冊到 `/demo/v1/user/token`，getInfo 註冊到 `/demo/v1/user/xxx/info`。後面的約束與 HttpSimpleController 的 PATH_ADD 巨集類似。

  若使用自動前綴，存取網址需包含命名空間和類名，本例需用 `http://localhost/demo/v1/user/token?userid=xxx&passwd=xxx` 或 `http://localhost/demo/v1/user/xxxxx/info?token=xxxx`。

  `ADD_METHOD_TO` 巨集與前者類似，但不會自動加前綴，註冊的路徑為絕對路徑。

  可見 `HttpController` 提供更彈性的路徑映射功能，且可註冊多個處理函式，方便將同類功能集中於一類。

  `METHOD_ADD` 巨集也提供參數映射方法，可將路徑參數映射到函式參數表，參數數字對應形參位置，常見可由字串型別轉換的型別都可作為參數（如 std::string、int、float、double 等），框架會自動型別推斷並轉換。注意左值引用必須為 const 型別。

  同一路徑可多次註冊，透過 HTTP method 區分，這是合法且常見的 Restful API 實作，例如：

  ```c++
  METHOD_LIST_BEGIN
      METHOD_ADD(Book::getInfo,"/{1}?detail={2}",Get);
      METHOD_ADD(Book::newBook,"/{1}",Post);
      METHOD_ADD(Book::deleteOne,"/{1}",Delete);
  METHOD_LIST_END
  ```

  路徑參數的占位符有多種寫法：

  * {}：表示此路徑參數映射到處理函式對應位置，路徑順序即參數順序。
  * {1}、{2}：中間數字表示映射到指定位置的參數。
  * {anystring}：中間字串僅為可讀性，與 {} 等價。
  * {1:anystring}、{2:xxx}：冒號前數字為位置，後字串僅為可讀性，與 {1}、{2} 等價。

  建議使用後兩種寫法，若路徑參數與函式參數順序一致，用第三種即可。以下幾種寫法等價：

  * "/users/{}/books/{}"
  * "/users/{}/books/{2}"
  * "/users/{user_id}/books/{book_id}"
  * "/users/{1:user_id}/books/{2}"

  > **注意：路徑比對不分大小寫，參數名稱區分大小寫，參數值保留原貌**

* #### 參數映射

  路徑參數和問號後的請求參數都可映射到處理函式參數表，目標參數型別需符合：

  * 必須是值型別、const 左值引用或非 const 右值引用之一，不能是非 const 左值引用，建議用右值引用，方便操作；
  * int、long、long long、unsigned long、unsigned long long、float、double、long double 等基本型別皆可；
  * std::string 型別；
  * 任何可用 `stringstream >>` 操作符賦值的型別；

  > **drogon 框架也支援從 HttpRequestPtr 物件到任意型別參數的映射機制**，若 handler 參數數量多於路徑參數，後面多餘參數會由 HttpRequestPtr 物件轉換，使用者可自訂型別轉換，只需特化 drogon 命名空間的 fromRequest 模板（定義於 HttpRequest.h），例如：

  ```c++
  namespace myapp{
  struct User{
      std::string userName;
      std::string email;
      std::string address;
  };
  }
  namespace drogon
  {
  template <>
  inline myapp::User fromRequest(const HttpRequest &req)
  {
      auto json = req.getJsonObject();
      myapp::User user;
      if(json)
      {
          user.userName = (*json)["name"].asString();
          user.email = (*json)["email"].asString();
          user.address = (*json)["address"].asString();
      }
      return user;
  }

  }
  ```

  有了上述定義與模板特化後，可如下定義路徑與 handler：

  ```c++
  class UserController:public drogon::HttpController<UserController>
  {
  public:
      METHOD_LIST_BEGIN
          //用 METHOD_ADD 新增自訂處理函式
          ADD_METHOD_TO(UserController::newUser,"/users",Post);
      METHOD_LIST_END
      // 處理函式宣告範例：
      void newUser(const HttpRequestPtr &req,
                  std::function<void (const HttpResponsePtr &)> &&callback,
                  myapp::User &&pNewUser) const;
  };
  ```

  可見第三個 `myapp::User` 型別參數在映射路徑上無對應占位符，框架會將其視為由 `req` 物件轉換的參數，透過使用者特化的模板取得，這些都由 drogon 透過模板推導在編譯期自動完成，極大便利開發。

  更進一步，若使用者只需自訂型別資料，不需存取 HttpRequestPtr 物件，也可將自訂物件放在第一個參數，框架同樣能正確映射，例如：

  ```c++
  class UserController:public drogon::HttpController<UserController>
  {
  public:
      METHOD_LIST_BEGIN
          //用 METHOD_ADD 新增自訂處理函式
          ADD_METHOD_TO(UserController::newUser,"/users",Post);
      METHOD_LIST_END
      // 處理函式宣告範例：
      void newUser(myapp::User &&pNewUser,
                  std::function<void (const HttpResponsePtr &)> &&callback) const;
  };
  ```

* #### 多路徑映射

  drogon 支援在路徑映射中使用正則表達式，在 `{}` 花括號以外部分可有限度使用，例如：

  ```c++
  ADD_METHOD_TO(UserController::handler1,"/users/.*",Post); /// 匹配所有以 `/users/` 開頭的路徑
  ADD_METHOD_TO(UserController::handler2,"/{name}/[0-9]+",Post); /// 匹配由名稱字串和數字組成的路徑
  ```

  此方法不支援子表達式、負向匹配等正則表達式，若需使用請參考下述方案。

* #### 正則表達式映射

  若需自由使用正則表達式，drogon 提供 `ADD_METHOD_VIA_REGEX` 巨集，例如：

  ```c++
  ADD_METHOD_VIA_REGEX(UserController::handler1,"/users/(.*)",Post); /// 匹配所有以 `/users/` 開頭的路徑，並將剩餘路徑映射到 handler1 參數
  ADD_METHOD_VIA_REGEX(UserController::handler2,"/.*([0-9]*)",Post); /// 匹配所有以數字結尾的路徑，並將該數字映射到 handler2 參數
  ADD_METHOD_VIA_REGEX(UserController::handler3,"/(?!data).*",Post); /// 匹配所有不以 '/data' 開頭的路徑
  ```

  使用正則表達式也可完成參數映射，所有子表達式匹配的字串會依序映射到 handler 參數。

  > **注意：使用正則表達式要注意匹配衝突（多個 handler 都匹配），若衝突發生在同一 controller 內，drogon 只執行第一個 handler（先註冊的）；若發生在不同 controller 間，執行哪個 handler 不確定，請避免此類衝突。**

# 下一步：[WebSocketController](/CHN/CHN-04-3-控制器-WebSocketController)

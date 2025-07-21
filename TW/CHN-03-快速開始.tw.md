[English](/ENG/ENG-03-Quick-Start)

# 靜態網站

我們從最簡單的範例開始介紹 drogon 的使用，這個範例會用指令工具 `drogon_ctl` 建立一個專案：

```bash
drogon_ctl create project your_project_name
```

進入專案目錄，可以看到以下檔案結構：

```console
├── build                         建構資料夾
├── CMakeLists.txt                專案的 cmake 設定檔
├── config.json                   drogon 應用程式設定檔
├── controllers                   控制器程式存放目錄
├── filters                       過濾器程式存放目錄
├── main.cc                       主程式
├── models                        資料庫模型存放目錄
│   └── model.json
└── views                         視圖 csp 檔案存放目錄
```

各資料夾名稱即代表用途，請將各類程式（如控制器、過濾器、視圖等）分別放入對應目錄，方便專案管理。詳細 `drogon_ctl` 用法請參考 [drogon_ctl](/CHN/CHN-11-drogon_ctl命令)。

main.cc 範例如下：

```c++
#include <drogon/HttpAppFramework.h>
int main() {
    // 設定 HTTP 監聽位址與埠號
    drogon::app().addListener("0.0.0.0",80);
    // 載入設定檔
    //drogon::app().loadConfigFile("../config.json");
    // 執行 HTTP 框架，此方法會在內部事件迴圈阻塞
    drogon::app().run();
    return 0;
}
```

接著建構專案：

```bash
cd build
cmake ..
make
```

編譯完成後，執行目標程式 `./your_project_name`。

現在，在 HTTP 根目錄新增一個最簡單的靜態檔案 index.html：

```bash
echo '<h1>Hello Drogon!</h1>' >>index.html
```

HTTP 根目錄預設值為 "./"，即 webapp 執行的目前路徑，也可在 config.json 設定檔中修改，詳情請參考 [設定檔](/CHN/CHN-10-配置文件)。然後在瀏覽器輸入 `http://localhost` 或 `http://localhost/index.html`（或你的 webapp 伺服器 IP）即可看到此頁面：
![Hello Drogon!](images/hellodrogon.png)

若伺服器找不到瀏覽器要求的頁面，會回傳 404 頁面：

![404頁面](images/notfound.png)

> **注意：請確認伺服器防火牆已開啟 80 埠，否則無法看到這些頁面（或將 port 改成 1024 以上以解決以下錯誤訊息）：**

```console
FATAL Permission denied (errno=13) , Bind address failed at 0.0.0.0:80 - Socket.cc:67
```

你可以將一個靜態網站的目錄與檔案複製到 webapp 執行目錄，然後用瀏覽器存取。drogon 預設支援的檔案類型有：

- html
- js
- css
- xml
- xsl
- txt
- svg
- ttf
- otf
- woff2
- woff
- eot
- png
- jpg
- jpeg
- gif
- bmp
- ico
- icns

drogon 也提供介面可自訂支援的檔案類型，詳情請參考 [HttpAppFramework 的 API](API-HttpAppFramework-中文)。

## 動態網站

接下來介紹如何為應用程式新增控制器（controller），並用控制器輸出內容。

在 `controller` 目錄下執行 drogon_ctl 指令工具產生控制器原始檔：

```bash
drogon_ctl create controller TestCtrl
```

會新增兩個檔案：TestCtrl.h 和 TestCtrl.cc

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
    // 在此撰寫應用邏輯
}
```

我們編輯這兩個檔案，讓控制器回應一個簡單的「Hello World!」。

TestCtrl.h 修改如下：

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
    //list path definitions here;
    //PATH_ADD("/path","filter1","filter2",HttpMethod1,HttpMethod2...);
    PATH_ADD("/",Get,Post);
    PATH_ADD("/test",Get);
    PATH_LIST_END
};
```

使用 PATH_ADD 將路徑映射到處理函式，這裡映射了 '/' 和 '/test'，並加上 HTTP 方法限制。

TestCtrl.cc 修改如下：

```c++
#include "TestCtrl.h"
void TestCtrl::asyncHandleHttpRequest(const HttpRequestPtr &req,
                                      std::function<void (const HttpResponsePtr &)> &&callback)
{
    // 應用邏輯
    //write your application logic here
    auto resp=HttpResponse::newHttpResponse();
    resp->setStatusCode(k200OK);
    resp->setContentTypeCode(CT_TEXT_HTML);
    resp->setBody("Hello World!");
    callback(resp);
}
```

重新用 cmake 編譯專案，然後執行目標程式 `./your_project_name`：

```bash
cd ../build
cmake ..
make
./your_project_name
```

在瀏覽器輸入 `http://localhost/` 或 `http://localhost/test`，即可看到 `Hello World!`。

> **注意：同時存在靜態與動態資源時，框架優先以控制器回應，此例中 `http://localhost/` 回應的是 `TestCtrl` 控制器的 `Hello World!`，而非靜態網頁 `index.html` 的 `Hello Drogon!`**

可見在應用程式中新增 controller 非常簡單，只需新增對應原始檔，甚至 main 檔案都不用修改，這種低耦合設計對 Web 應用開發非常有效。

> **注意：Drogon 沒有限制控制器原始檔的位置，也可放在專案目錄下，甚至可在 `CMakeLists.txt` 指定新目錄。為方便管理，建議將控制器原始檔放在 controllers 目錄。**

# 下一步：[控制器簡介](/CHN/CHN-04-0-控制器-簡介)

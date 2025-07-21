[English](/ENG/ENG-06-View)

# 視圖介紹

雖然目前前端渲染技術盛行，後端應用服務通常只需回傳資料給前端，但一個好的 Web 框架仍應提供後端渲染技術，讓伺服器端程式能動態產生 HTML 頁面。視圖（View）可協助使用者產生這些頁面，顧名思義，它只負責展示相關工作，複雜業務邏輯則交由控制器處理。

早期 Web 應用程式常將 HTML 嵌入程式碼以動態產生頁面，但這種方式效率低且不直觀。後來如 JSP 等語言反其道而行，將程式碼嵌入 HTML 頁面。drogon 採用的也是後者，但因 C++ 為編譯型語言，嵌入 C++ 程式碼的頁面需先轉換成 C++ 原始檔才能編譯進應用程式。因此 drogon 定義了專屬的 CSP（C++ Server Pages）描述語言，可用指令工具 `drogon_ctl` 將 csp 檔案轉換成 C++ 原始檔供編譯。

### Drogon 的 csp

drogon 的 csp 方案很簡單，只需用特殊標記將 C++ 程式碼嵌入 HTML 頁面：

* 夾在 `<%inc` 和 `%>` 之間的內容視為需引用的標頭檔，只能寫 `#include` 語句，如 `<%inc#include "xx.h" %>`，但多數常用標頭檔 drogon 已自動包含，基本用不到此標籤；
* 夾在 `<%c++` 和 `%>` 之間的內容視為 C++ 程式碼，如 `<c++ std:string name="drogon"; %>`；
* C++ 程式碼一般會原封不動轉移到目標原始檔，除了以下兩種特殊標記：
  * `@@` 代表控制器傳入的 data 變數，型別為 `HttpViewData`，可取得所需內容；
  * `$$` 代表頁面內容的流物件，可用 `<<` 操作符將要顯示的內容輸出到頁面；
* 夾在 `[[` 和 `]]` 之間的內容視為變數名稱，view 會以此名稱為 keyword 從控制器傳入的資料中找到對應變數並輸出到頁面，變數名稱前後空白會被省略，`[[` 和 `]]` 不要分行寫。僅支援三種字串型別（const char *、std::string、const std::string），其他型別請用上述方式輸出（或將變數以 string 型別存入 data）；
* 夾在 `{%` 和 `%}` 之間的內容視為 C++ 程式內變數名稱或運算式（非控制器傳入資料的 keyword），view 會將該變數或運算式的值輸出到頁面。`{%val.xx%}` 等同 `<%c++$$<<val.xx;%>`，但前者更簡潔。兩標籤勿分行；
* 夾在 `<%view` 和 `%>` 之間的內容視為子視圖名稱，框架會找到相應子視圖並將其內容填入該位置；視圖名稱前後空白會被忽略，`<%view` 和 `%>` 不要分行寫，子視圖與父視圖共用控制器資料，可多層嵌套但勿循環嵌套。
* 夾在 `<%layout` 和 `%>` 之間的內容視為版型名稱，框架會找到相應版型並將本視圖內容填入版型指定位置（由佔位符 `[[]]` 標示）；版型名稱前後空白會被忽略，`<%layout` 和 `%>` 不要分行寫，可多層嵌套但勿循環嵌套。

### 視圖的使用

drogon 應用程式的 HTTP 回應皆由控制器 handler 產生，視圖渲染的回應也由 handler 產生，可用以下介面：

```c++
static HttpResponsePtr newHttpViewResponse(const std::string &viewName,
                                           const HttpViewData &data);
```

此介面為 HttpResponse 類的靜態方法，兩個參數：

* **viewName**：視圖名稱，傳入 csp 檔名（副檔名可省略）；
* **data**：控制器 handler 傳給視圖的資料，型別為 `HttpViewData`，可存取任意型別物件，詳見 [HttpViewData API](API-HttpViewData-中文)；

控制器無需引用視圖標頭檔，控制器與視圖高度解耦，唯一聯繫為 data 變數，控制器與視圖需對 data 內容有一致約定。

### 簡單範例

以下範例將瀏覽器送來的 HTTP 請求參數顯示在回傳的 HTML 頁面。

直接用 HttpAppFramework 介面定義 handler，在 main 檔 run() 前加入：

```c++
drogon::HttpAppFramework::instance()
 .registerHandler
  ("/list_para",
   [=](const HttpRequestPtr &req,
       std::function<void (const HttpResponsePtr &)> &&callback)
       {
            auto para=req->getParameters();
            HttpViewData data;
            data.insert("title","ListParameters");
            data.insert("parameters",para);
            auto resp=HttpResponse::newHttpViewResponse("ListParameters.csp",data);
            callback(resp);
        }
   );
```

上述程式將 lambda handler 註冊到 `/list_para` 路徑，取得請求參數並傳給視圖顯示。
然後進入 views 資料夾，建立視圖檔 ListParameters.csp，內容如下：

```html
<!DOCTYPE html>
<html>
<%c++
    auto para=@@.get<std::unordered_map<std::string,std::string,utils::internal::SafeStringHash>>("parameters");
%>
<head>
    <meta charset="UTF-8">
    <title>[[ title ]]</title>
</head>
<body>
    <%c++ if(para.size()>0){%>
    <H1>Parameters</H1>
    <table border="1">
      <tr>
        <th>name</th>
        <th>value</th>
      </tr>
      <%c++ for(auto iter:para){%>
      <tr>
        <td>{%iter.first%}</td>
        <td><%c++ $$<<iter.second;%></td>
      </tr>
      <%c++}%>
    </table>
    <%c++ }else{%>
    <H1>no parameter</H1>
    <%c++}%>
</body>
</html>
```

可用以下指令將 ListParameters.csp 轉換成 C++ 原始檔：

```bash
drogon_ctl create view ListParameters.csp
```

執行後會產生 ListParameters.h 和 ListParameters.cc 原始檔，可編譯進 web 應用程式。

用 cmake 重新編譯整個專案，執行 webapp，於瀏覽器輸入 `http://localhost/list_para?p1=a&p2=b&p3=c`，即可看到如下頁面：

![view頁面](images/viewdemo.png)

後端渲染的 HTML 頁面就這麼簡單加上了。雖然頁面較簡陋，但不影響說明視圖用法。

### csp 檔案自動化處理

> **注意：若專案是用 `drogon_ctl` 指令建立，則本節內容已由該指令自動處理。**

顯然，每次修改 csp 檔都需手動執行 drogon_ctl 指令不太方便，可將 drogon_ctl 處理寫進 CMakeLists.txt。以本範例為例，假設所有 csp 檔都放在 views 資料夾，則 CMakeLists.txt 可加上：

```cmake
FILE(GLOB SCP_LIST ${CMAKE_CURRENT_SOURCE_DIR}/views/*.csp)
foreach(cspFile ${SCP_LIST})
    message(STATUS "cspFile:" ${cspFile})
    EXEC_PROGRAM(basename ARGS "-s .csp ${cspFile}" OUTPUT_VARIABLE classname)
    message(STATUS "view classname:" ${classname})
    add_custom_command(OUTPUT ${classname}.h ${classname}.cc
        COMMAND drogon_ctl
        ARGS create view ${cspFile}
        DEPENDS ${cspFile}
        VERBATIM )
   set(VIEWSRC ${VIEWSRC} ${classname}.cc)
endforeach()
```

然後在 add_executable 語句加入新原始檔集合 ${VIEWSRC}，如下：

```cmake
add_executable(webapp ${SRC_DIR} ${VIEWSRC})
```

上述作法在 `drogon_ctl create project` 產生的專案已寫入 CMakeLists.txt，使用者在 views 資料夾建立的 csp 檔都會自動轉換並編譯進應用程式。

### 視圖的動態編譯與載入

drogon 提供應用程式執行期動態編譯與載入 csp 檔的方法，可用以下介面設定：

```c++
void enableDynamicViewsLoading(const std::vector<std::string> &libPaths);
```

此介面為 `HttpAppFramework` 成員方法，參數為字串陣列，代表視圖 csp 檔所在目錄列表。呼叫後，drogon 會自動搜尋這些目錄，發現新或被修改的 csp 檔即自動產生原始檔、編譯成動態庫並載入，整個過程無需重啟應用程式。可自行實驗，觀察 csp 修改後頁面變化。

此功能依賴開發環境，若 drogon 與 webapp 都在同台伺服器編譯，則動態載入 csp 頁面應無問題。

> **注意：動態載入的視圖不能靜態編譯進程式，若某視圖已靜態編譯，則無法透過動態載入更新。可另建動態視圖路徑，開發階段將視圖移至該路徑調試（Linux 無此問題）。**

> **注意：此特性建議用於開發階段方便調整頁面，正式環境仍建議直接編譯成目標檔運行，主要考量安全性與穩定性。**

> **注意：若載入時遇到 `symbol not found` 錯誤，請用 `cmake .. -DCMAKE_ENABLE_EXPORTS=on` 或取消 CMakeLists.txt 最後一行 `set_property(TARGET ${PROJECT_NAME} PROPERTY ENABLE_EXPORTS ON)` 的註解，並重新編譯專案**

# 下一步：[會話](/CHN/CHN-07-會話)

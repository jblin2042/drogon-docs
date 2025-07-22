[English](/ENG/ENG-11-Configuration-File) 

# 設定檔

你可以透過 DrogonAppFramework 實例的多種介面設定參數，控制 Http 伺服器行為。不過，使用設定檔更佳，原因如下：

* 設定檔而非原始碼，可於執行期而非編譯期決定應用行為，更方便彈性；
* 可保持 main 檔案簡潔；

所有設定介面皆有對應設定檔選項，建議開發者以設定檔管理各項參數。

載入設定檔很簡單，於 DrogonAppFramework 實例呼叫 run 前，呼叫 loadConfigFile，參數為設定檔路徑與檔名，例如：

```c++
int main()
{
    drogon::app().loadConfigFile("config.json");
    drogon::app().run();
}
```

此程式載入 `config.json`，再執行。監聽埠、日誌、資料庫等皆可由設定檔管理，這段程式基本可作為整個 webapp 主函式。

## 檔案說明

設定檔範例見原始碼目錄頂層 `config.example.json`，用 drogon_ctl create project 建立專案時也會有同內容的 `config.json`。基本只需修改此檔即可完成 webapp 設定。

檔案為 json 格式，支援註解，可用 C++ 註解符號 `/**/` 與 `//`。

註解掉設定項時，框架會用預設值初始化，預設值皆於設定檔中說明。

## 支援格式
* json
* yaml（需安裝 yaml-cpp）

以下以 json 格式分項說明。

### SSL

ssl 項用於設定 https 加密：

```json
  "ssl": {
    "cert": "../../trantor/trantor/tests/server.pem",
    "key": "../../trantor/trantor/tests/server.pem",
    "conf":[
      ["Options", "Compression"],
      ["min_protocol", "TLSv1.2"]
    ]
  }
```

cert 為憑證路徑，key 為私鑰路徑，若同檔案含憑證與私鑰則可設相同。檔案格式為 PEM；conf 為 SSL 選項，會直接傳入 [SSL_CONF_cmd](https://www.openssl.org/docs/manmaster/man3/SSL_CONF_cmd.html) 設定 SSL 連線，必須為一或兩元素字串陣列。

### listeners 監聽器

listeners 用於設定 webapp 監聽器，為 JSON 陣列，每個物件為一監聽設定：

```json
"listeners": [
    {
      "address": "0.0.0.0",
      "port": 80,
      "https": false
    },
    {
      "address": "0.0.0.0",
      "port": 443,
      "https": true,
      "cert": "",
      "key": ""
    }
  ]
```

* `address`：字串，監聽 IP，預設 "0.0.0.0"
* `port`：整數，監聽埠，必填
* `https`：布林，是否用 https，預設 false
* `cert`、`key`：字串，https 憑證與私鑰，預設空字串，表示用全域 ssl 設定

### db_clients 資料庫客戶端

設定資料庫客戶端，為 JSON 陣列，每個物件為一客戶端設定：

```json
  "db_clients":[
    {
      "name":"",
      "rdbms": "postgresql",
      "host": "127.0.0.1",
      "port": 5432,
      "dbname": "test",
      "user": "",
      "passwd": "",
      "is_fast": false,
      "connection_number":1,
      "filename": ""
    }
  ]
```

* `name`：字串，客戶端名稱，預設 "default"，多客戶端時 name 必不同
* `rdbms`：字串，資料庫型別，支援 "postgresql"、"mysql"、"sqlite3"
* `host`：字串，資料庫地址，預設 localhost
* `port`：整數，資料庫埠
* `dbname`：字串，資料庫名稱
* `user`：字串，使用者名稱
* `passwd`：字串，密碼
* `is_fast`：布林，預設 false，是否為 [FastDbClient](/CHN/CHN-08-4-数据库-FastDbClient)
* `connection_number`：連線數，至少 1，預設 1，影響併發量；is_fast 為 true 時表示每個事件循環連線數，否則為總連線數
* `filename`：sqlite3 資料庫檔名

### threads_num 執行緒數

app 子項，整數，預設 1，表示框架 IO 執行緒數，影響網路併發。此數值應與期望網路 IO 佔用 CPU 核心數一致。設 0 則用全部 CPU 核心數。

```json
"threads_num": 16,
```

### session 會話

app 子項，控制是否啟用 session 及超時：

```json
"enable_session": true,
"session_timeout": 1200,
```

* `enable_session`：布林，是否啟用 session，預設 false。若客戶端不支援 cookie 請設 false，否則會產生大量無用 session。
* `session_timeout`：整數，session 超時秒數，預設 0，僅 enable_session 為 true 時有效。0 表示永久有效。

### document_root 根目錄

app 子項，字串，Http 根目錄路徑，靜態檔案下載根路徑，預設 "./"（程式執行目錄）。

```json
"document_root": "./",
```

### upload_path 上傳檔案路徑

app 子項，字串，上傳檔案預設路徑，預設 "uploads"。若非 `/`、`./`、`../` 開頭，且非 `.`、`..`，則為 document_root 相對路徑，否則為絕對路徑或目前目錄相對路徑。

```json
"upload_path":"uploads",
```

### client_max_body_size 請求體最大值

app 子項，字串，限制請求體大小。可用後綴 `k`、`m`、`g`、`t` 指定單位，大小寫皆可。

```json
"client_max_body_size": "10M",
```

### client_max_memory_body_size 請求體記憶體分配最大值

app 子項，字串，限制請求體快取分配記憶體大小。用法同上。

```json
"client_max_memory_body_size": "50K"
```

### file_types 檔案類型

app 子項，字串陣列，預設如下，表示支援下載的靜態檔案類型，若請求檔案副檔名不在此列，框架回傳 404。

```json
"file_types": [
      "gif",
      "png",
      "jpg",
      "js",
      "css",
      "html",
      "ico",
      "swf",
      "xap",
      "apk",
      "cur",
      "xml"
    ],
```

### mime 註冊新 MIME 類型

app 子項，字串字典。宣告靜態檔案副檔名對應新 MIME 類型。僅註冊 MIME，若副檔名不在 file_types，仍回傳 404。

```json
"mime" : {
  "text/markdown": "md",
  "text/gemini": ["gmi", "gemini"]
}
```

### connections 連線數控制

app 子項，有兩選項：

```json
"max_connections": 100000,
"max_connections_per_ip": 0,
```

* `max_connections`：整數，預設 100000，最大同時連線數，達上限新連線會被拒絕
* `max_connections_per_ip`：整數，預設 0，單一 IP 最大連線數，0 表示無限制

### log 日誌選項

app 子項，JSON 物件，控制日誌輸出：

```json
"log": {
      "log_path": "./",
      "logfile_base_name": "",
      "log_size_limit": 100000000,
      "log_level": "TRACE"
    },
```

* `log_path`：字串，預設空，表示檔案路徑，空則輸出至標準輸出
* `logfile_base_name`：字串，日誌檔案 basename，預設空，則為 drogon
* `log_size_limit`：整數，單位位元組，預設 100000000 (100M)，達上限時切換檔案
* `log_level`：字串，預設 "DEBUG"，最低日誌級別，選項："TRACE"、"DEBUG"、"INFO"、"WARN"，TRACE 僅 DEBUG 編譯有效

> **注意：Drogon 檔案日誌採非阻塞結構，可達每秒百萬行日誌輸出，請放心使用。**

### 應用控制

app 子項，有兩控制項：

```json
    "run_as_daemon": false,
    "relaunch_on_error": false,
```

* `run_as_daemon`：布林，預設 false，true 時應用以 daemon 方式於系統背景執行
* `relaunch_on_error`：布林，預設 false，true 時應用 fork 一次，子行程執行主程式，父行程監控並於子行程崩潰時重啟，為簡易服務保護機制

### use_sendfile 傳送檔案

app 子項，布林，表示傳送檔案時是否用 linux sendfile，預設 true。sendfile 可提升效率、減少大檔案記憶體占用。

```json
"use_sendfile": true,
```

> **注意：即使設 true，sendfile 不一定採用，框架會依最佳化策略決定。**

### use_gzip 壓縮傳輸

app 子項，布林，預設 true，表示 Http 回應 Body 是否壓縮。true 時，以下情況採壓縮：

* 客戶端支援 gzip
* body 為文字型態
* body 長度大於一定值

範例：

```json
"use_gzip": true,
```

### static_files_cache_time 檔案快取時間

app 子項，整數，單位秒，靜態檔案於框架內快取時間。期間內請求直接由記憶體回應，不讀檔案系統。預設 5 秒，0 表示永久快取（僅讀檔案一次，慎用），負值表示無快取。

```json
"static_files_cache_time": 5,
```

### simple_controllers_map 簡易控制器映射

app 子項，JSON 物件陣列，每項為 Http 路徑至 HttpSimpleController 映射。此設定為選用，詳見 [HttpSimpleController](/CHN/CHN-04-1-控制器-HttpSimpleController)。

```json
"simple_controllers_map": [
      {
        "path": "/path/name",
        "controller": "controllerClassName",
        "http_methods": ["get","post"],
        "filters": ["FilterClassName"]
      }
    ],
```

* `path`：字串，Http 路徑
* `controller`：字串，HttpSimpleController 名稱
* `http_methods`：字串陣列，支援 Http 方法，其他方法回傳 405
* `filters`：字串陣列，路徑上的 filter，詳見 [中介軟體與過濾器](/CHN/CHN-05-中间件和过滤器)

### idle_connection_timeout 閒置連線逾時

app 子項，整數，單位秒，預設 60。連線超過此時間無讀寫即強制斷線。

```json
"idle_connection_timeout":60
```

### dynamic_views 動態視圖載入

app 子項，控制動態視圖啟用與路徑，有兩選項：

```json
"load_dynamic_views":true,
"dynamic_views_path":["./views"],
```

* `load_dynamic_views`：布林，預設 false，true 時框架會於視圖路徑搜尋並動態編譯 so 檔後載入，視圖檔變更時自動編譯與重載
* `dynamic_views_path`：字串陣列，動態視圖搜尋路徑，若非 `/`、`./`、`../` 開頭且非 `.`、`..`，則為 document_root 相對路徑，否則為絕對路徑或目前目錄相對路徑

詳見 [視圖](/CHN/CHN-06-视图)。

### server_header_field 標頭欄位

app 子項，設定所有 response 的 Server 標頭欄位，預設空字串，空時自動產生 `Server: drogon/version string` 標頭。

```json
"server_header_field": ""
```

### keepalive_requests 長連線請求數

`keepalive_requests` 設定客戶端於一 keepalive 長連線可送出最大請求數，達上限即關閉。預設 0 表示無限制。

```json
"keepalive_requests": 0
```

### Pipelining 請求數

`pipelining_requests` 設定長連線上已接收但未處理的最大請求數，達上限即關閉。預設 0 表示無限制。詳見 RFC2616-8.1.1.2。

```json
"pipelining_requests": 0
```

# 下一步：[drogon_ctl 指令](/CHN/CHN-12-drogon_ctl命令)

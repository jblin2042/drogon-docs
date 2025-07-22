[English](/ENG/ENG-12-drogon_ctl-Command)

# drogon_ctl

**Drogon** 框架編譯安裝後，會同時安裝一個命令列工具 `drogon_ctl`，另有一個同功能副本 `dg_ctl`，可依喜好選用。

此工具主要用於協助使用者建立各種 drogon 專案檔案，執行 `dg_ctl help` 可查看支援功能：

```console
$ dg_ctl help
usage: drogon_ctl <command> [<args>]
commands list:
create                  create some source files(Use 'drogon_ctl help create' for more information)
help                    display this message
version                 display version of this tool
press                   Do stress testing(Use 'drogon_ctl help press' for more information)
```

## version 子命令

version 用於顯示目前安裝於系統的 drogon 版本：

```console
$ dg_ctl version
     _
  __| |_ __ ___   __ _  ___  _ __
 / _` | '__/ _ \ / _` |/ _ \| '_ \
| (_| | | | (_) | (_| | (_) | | | |
 \__,_|_|  \___/ \__, |\___/|_| |_|
                 |___/

drogon ctl tools
version:0.9.30.771
git commit:d4710d3da7ca9e73b881cbae3149c3a570da8de4
compile config:-O3 -DNDEBUG -Wall -std=c++17 -I/root/drogon/trantor -I/root/drogon/lib/inc -I/root/drogon/orm_lib/inc -I/usr/local/include -I/usr/include/uuid -I/usr/include -I/usr/include/mysql
```

## create 子命令

create 用於建立各種物件，目前是 drogon_ctl 的主要功能。執行 `dg_ctl help create` 可查看詳細說明：

```console
$ dg_ctl help create
Use create command to create some source files of drogon webapp

Usage:drogon_ctl create <view|controller|filter|project|model> [-options] <object name>

drogon_ctl create view <csp file name> //create HttpView source files from csp file

drogon_ctl create controller [-s] <[namespace::]class_name> //create HttpSimpleController source files

drogon_ctl create controller -h <[namespace::]class_name> //create HttpController source files

drogon_ctl create controller -w <[namespace::]class_name> //create WebSocketController source files

drogon_ctl create filter <[namespace::]class_name> //create a filter named class_name

drogon_ctl create project <project_name> //create a project named project_name

drogon_ctl create model <model_path> //create model classes in model_path
```

* ### 視圖建立

  `dg_ctl create view` 用於由 csp 檔案產生原始碼，詳見[視圖](/CHN/CHN-06-视图)。一般不需直接使用，建議於 cmake 設定自動執行。範例：

  ```shell
  dg_ctl create view UsersList.csp
  ```

* ### 控制器建立

  `dg_ctl create controller` 用於建立控制器原始碼，支援三種控制器，參數略有不同。

  * 建立 HttpSimpleController：

  ```shell
  dg_ctl create controller SimpleControllerTest
  dg_ctl create controller webapp::v1::SimpleControllerTest
  ```

  最後參數為類名，可加命名空間。

  * 建立 HttpController：

  ```shell
  dg_ctl create controller -h ControllerTest
  dg_ctl create controller -h api::v1::ControllerTest
  ```

  * 建立 WebSocketController：

  ```shell
  dg_ctl create controller -w WsControllerTest
  dg_ctl create controller -w api::v1::WsControllerTest
  ```

  詳見 wiki 各控制器章節。

* ### 過濾器建立

  `dg_ctl create filter` 用於建立過濾器原始碼，詳見[中介軟體與過濾器](/CHN/CHN-05-中间件和过滤器)。

  ```shell
  dg_ctl create filter LoginFilter
  dg_ctl create filter webapp::v1::LoginFilter
  ```

* ### 專案建立

  建立新 Drogon 應用專案：

  ```shell
  dg_ctl create project ProjectName
  ```

  執行後會於目前目錄建立完整專案目錄，名稱為 `ProjectName`，可直接進 build 目錄編譯，預設無業務邏輯。

  專案結構如下：

  ```console
  ├── build                         建構目錄
  ├── CMakeLists.txt                cmake 設定檔
  ├── cmake_modules                 第三方庫查找 cmake 腳本
  │   ├── FindJsoncpp.cmake
  │   ├── FindMySQL.cmake
  │   ├── FindSQLite3.cmake
  │   └── FindUUID.cmake
  ├── config.json                   drogon 設定檔，詳見設定檔章節
  ├── controllers                   控制器原始碼目錄
  ├── filters                       過濾器原始碼目錄
  ├── main.cc                       主程式
  ├── models                        資料庫模型目錄，模型建立見 ORM 章節
  │   └── model.json
  ├── tests                         測試程式目錄
  │   └── test_main.cc              測試主程式
  └── views                         視圖 csp 檔目錄，原始碼由編譯時自動產生
  ```

* ### 模型建立

  使用 `dg_ctl create model` 建立資料庫模型原始碼，最後參數為模型目錄，該目錄需含 `model.json` 設定檔，告知 dg_ctl 如何連線資料庫及映射哪些表。

  例如於上述專案目錄建立模型：

  ```shell
  dg_ctl create model models
  ```

  執行後會提示目錄下既有檔案將被覆蓋，輸入 `y` 確認後即產生所有模型檔案。引用模型類時需 include 模型標頭檔，例如：

  ```c++
  #include "models/User.h"
  ```

  注意需包含 models 目錄名，以區分多資料源。詳見[ORM](/CHN/CHN-08-3-数据库-ORM)。

## 壓力測試

可用 `dg_ctl press` 進行壓力測試，選項如下：

* `-n num` 設定總請求數（預設 1）
* `-t num` 設定執行緒數（預設 1），設為 CPU 數可達最大效能
* `-c num` 設定併發連線數（預設 1）
* `-q` 不輸出中間過程資訊（預設輸出）

範例：

```shell
dg_ctl press -n1000000 -t4 -c1000 -q http://localhost:8080/
dg_ctl press -n 1000000 -t 4 -c 1000 https://www.domain.com/path/to/be/tested
```

# 下一步：[AOP 面向切面程式設計](/CHN/CHN-13-AOP面向切面编程)

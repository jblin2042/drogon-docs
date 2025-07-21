# 安裝 Drogon

[English](/ENG/ENG-02-Installation)

本章以 Ubuntu 24.04、CentOS 7.5、macOS 12.2 為例，簡介安裝流程，其他作業系統大同小異。

## 系統需求

* Linux 核心版本需不低於 2.6.9，64 位元版本；
* gcc 版本不低於 5.4.0，建議使用 11 以上版本；
* 建構工具為 cmake，cmake 版本需不低於 3.5；
* git 版本管理工具；

## 相依套件

* 內建
  * trantor，非阻塞 I/O C++ 網路函式庫，已作為 git 子模組，無需事先安裝；
* 必須
  * jsoncpp，json 的 C++ 函式庫，版本**不低於 1.7**；
  * libuuid，產生 uuid 的 C 函式庫；
  * zlib，用於支援壓縮傳輸；
* 選用
  * boost，版本**不低於 1.61**，僅在 C++ 編譯器不支援 c++17 或 STL 函式庫不完整支援 `std::filesystem` 時才需安裝；
  * OpenSSL，安裝後 drogon 將支援 HTTPS，否則僅支援 HTTP；
  * c-ares，安裝後 drogon 對 DNS 的支援會更好；
  * libbrotli，安裝後 drogon 的 HTTP 回應會支援 brotli 壓縮；
  * postgreSQL、mariadb、sqlite3 的客戶端開發函式庫，安裝後 drogon 會提供對應資料庫的存取能力；
  * hiredis，安裝後 drogon 將支援 redis 存取；
  * gtest，安裝後 drogon 的單元測試程式碼可編譯；
  * yaml-cpp，安裝後 drogon 將支援 yaml 格式的設定檔；

## 系統準備範例

#### Ubuntu 24.04

* 環境

  ```bash
  sudo apt install git gcc g++ cmake
  ```

* jsoncpp

  ```bash
  sudo apt install libjsoncpp-dev
  ```

* uuid

  ```bash
  sudo apt install uuid-dev
  ```

* zlib

  ```bash
  sudo apt install zlib1g-dev
  ```

* OpenSSL（選用，提供 HTTPS 支援）

  ```bash
  sudo apt install openssl libssl-dev
  ```

#### CentOS 7.5

* 環境

  ```bash
  yum install git
  yum install gcc
  yum install gcc-c++
  ```

  # 預設安裝的 cmake 版本太舊，請用原始碼安裝

  ```bash
  git clone https://github.com/Kitware/CMake
  cd CMake/
  ./bootstrap && make && make install
  ```

  # 升級 gcc

  ```bash
  yum install centos-release-scl
  yum install devtoolset-11
  scl enable devtoolset-11 bash
  ```

  > 注意：`scl enable devtoolset-11 bash` 指令僅為暫時啟用新版 gcc，直到終端關閉。若要永久使用新版 gcc，可用 `echo "scl enable devtoolset-11 bash" >> ~/.bash_profile`，系統重啟後將自動啟用新版 gcc。

* jsoncpp

  ```bash
  git clone https://github.com/open-source-parsers/jsoncpp
  cd jsoncpp/
  mkdir build
  cd build
  cmake ..
  make && make install
  ```

* uuid

  ```bash
  yum install libuuid-devel
  ```

* zlib

  ```bash
  yum install zlib-devel
  ```

* OpenSSL（選用，提供 HTTPS 支援）

  ```bash
  yum install openssl-devel
  ```

#### macOS 12.2

* 環境

  macOS 內建皆有，更新即可

  ```bash
  brew upgrade
  ```

* jsoncpp

  ```bash
  brew install jsoncpp
  ```

* uuid

  ```bash
  brew install ossp-uuid
  ```

* zlib

  ```bash
  brew install zlib
  ```

* OpenSSL（選用，提供 HTTPS 支援）

  ```bash
  brew install openssl
  ```

#### Windows

* 環境：

  安裝 Visual Studio 2019 專業版，安裝選項中至少包含：

  * MSVC C++ 編譯工具
  * Windows 10 SDK
  * 用於 Windows 的 C++ CMake 工具
  * Google Test 測試配套

`conan` 套件管理工具可提供 Drogon 專案的所有相依，若有 python 環境，可用 pip 安裝 `conan` 套件管理工具。

```bash
pip install conan
```

> 當然也可從 [官網](https://conan.io/) 下載 `conan` 安裝檔進行安裝。

建立 `conanfile.txt` 檔案並加入如下內容：

* jsoncpp

  ```txt
  [requires]
  jsoncpp/1.9.4
  ```

* uuid

  不需安裝，Windows 10 SDK 已包含 uuid 函式庫。

* zlib

  ```txt
  [requires]
  zlib/1.2.11
  ```

* OpenSSL（選用，提供 HTTPS 支援）

  ```txt
  [requires]
  openssl/1.1.1t
  ```

## 資料庫環境（選用）

> 注意：以下資料庫皆非必須，使用者可依實際需求選擇安裝一種或多種資料庫。

> 注意：若未來開發需要用到資料庫，請先安裝好資料庫環境再安裝 drogon，否則會出現找不到資料庫的問題。

* #### PostgreSQL

  PostgreSQL 原生 C 函式庫 libpq 需安裝，安裝方式如下：

  * `ubuntu 16`: `sudo apt-get install postgresql-server-dev-all`
  * `ubuntu 18`: `sudo apt-get install postgresql-all`
  * `centOS 7`: `yum install postgresql-devel`
  * `macOS`: `brew install postgresql`
  * `Windows conanfile`: `libpq/13.2`

* #### MySQL

  MySQL 原生函式庫不支援非同步讀寫，Drogon 採用 MariaDB 開發函式庫，MariaDB 與 MySQL 相容且支援非同步，建議統一安裝 MariaDB。

  安裝方式如下：

  * `ubuntu`: `sudo apt install libmariadbclient-dev`
  * `centOS 7`: `yum install mariadb-devel`
  * `macOS`: `brew install mariadb`
  * `Windows conanfile`: `libmariadb/3.1.13`

* #### Sqlite3

  * `ubuntu`: `sudo apt-get install libsqlite3-dev`
  * `centOS`: `yum install sqlite-devel`
  * `macOS`: `brew install sqlite3`
  * `Windows conanfile`: `sqlite3/3.36.0`

* #### Redis

  * `ubuntu`: `sudo apt-get install libhiredis-dev`
  * `centOS`: `yum install hiredis-devel`
  * `macOS`: `brew install hiredis`
  * `Windows conanfile`: `hiredis/1.0.0`

> 注意：上述有些指令僅安裝了開發函式庫，若還要安裝 server 端，請自行查詢。

## 安裝 Drogon

假設上述系統環境與函式庫相依都已準備好，安裝流程非常簡單：

* #### Linux 原始碼安裝

  ```bash
  cd $WORK_PATH
  git clone https://github.com/drogonframework/drogon
  cd drogon
  git submodule update --init
  mkdir build
  cd build
  cmake ..
  make && sudo make install
  ```

  > 預設編譯為 debug 版本，若要編譯 release 版本，cmake 指令需加上如下參數：

  ```bash
  cmake -DCMAKE_BUILD_TYPE=Release ..
  ```

  安裝結束後，會有如下檔案安裝在系統中（CMAKE_INSTALL_PREFIX 可變更安裝位置）：

  * drogon 的標頭檔安裝到 /usr/local/include/drogon
  * drogon 的函式庫檔案 libdrogon.a 安裝到 /usr/local/lib
  * drogon 的指令工具 drogon_ctl 安裝到 /usr/local/bin
  * trantor 的標頭檔安裝到 /usr/local/include/trantor
  * trantor 的函式庫檔案 libtrantor.a 安裝到 /usr/local/lib

* #### Windows 原始碼安裝

  1. 下載 Drogon 原始碼

      開啟 Windows 任務列搜尋框，搜尋 x64 Native Tools，選擇 x64 Native Tools Command Prompt for VS 2019 作為指令工具；

      ```bash
      cd $WORK_PATH
      git clone https://github.com/drogonframework/drogon
      cd drogon
      git submodule update --init
      ```

  2. 安裝相依函式庫

      ```bash
      mkdir build
      cd build
      conan profile detect --force
      conan install .. -s compiler="msvc" -s compiler.version=193 -s compiler.cppstd=17 -s build_type=Debug  --output-folder . --build=missing
      ```

     > 編輯 `conanfile.txt` 檔案可加入相依，修改函式庫版本。

  3. 編譯並安裝

      ```bash
      cmake ..  -DCMAKE_BUILD_TYPE=Debug -DCMAKE_TOOLCHAIN_FILE="conan_toolchain.cmake" -DCMAKE_POLICY_DEFAULT_CMP0091=NEW -DCMAKE_INSTALL_PREFIX="D:"
      cmake --build . --parallel --target install
      ```

  > 注意：conan 與 cmake 的 build type 必須一致。

  安裝結束後，會有如下檔案安裝在系統中（CMAKE_INSTALL_PREFIX 可變更安裝位置）：

  * drogon 的標頭檔安裝到 `D:/include/drogon`
  * drogon 的函式庫檔案 drogon.dll 安裝到 `D:/bin`
  * drogon 的指令工具 drogon_ctl.exe 安裝到 `D:/bin`
  * trantor 的標頭檔安裝到 `D:/include/trantor`
  * trantor 的函式庫檔案 trantor.dll 安裝到 `D:/bin`

  加入 `bin` 與 `cmake` 路徑到 `path` 環境變數：

  ```
  D:\bin
  D:\lib\cmake\Drogon
  D:\lib\cmake\Trantor
  ```

* #### Windows vcpkg 安裝

  [安裝教學](https://www.youtube.com/watch?v=0ojHvu0Is6A)

  **安裝 vcpkg：**
  1. 用 git 安裝 vcpkg。

     ```bash
     git clone https://github.com/microsoft/vcpkg
     cd vcpkg
     ./bootstrap-vcpkg.bat
     ```

     > 說明：要升級 vcpkg，只需輸入 `git pull`

  2. 將 vcpkg 路徑加入環境變數 path。
  3. 終端輸入 `vcpkg` 或 `vcpkg.exe` 檢查 vcpkg 是否安裝成功。

  **正式安裝 Drogon：**

  1. 輸入指令安裝 drogon 框架：

     * 32 位元：`vcpkg install drogon`
     * 64 位元：`vcpkg install drogon:x64-windows`
     * 進階：`vcpkg install jsoncpp:x64-windows zlib:x64-windows openssl:x64-windows sqlite3:x64-windows libpq:x64-windows libpqxx:x64-windows drogon[core,ctl,sqlite3,postgres,orm]:x64-windows`

     注意：

     * 若有相依套件未安裝而出錯，只需安裝該套件，例如：

       zlib：`vcpkg install zlib` 或 `vcpkg install zlib:x64-windows`（64 位元）

     * 檢查安裝結果：

       `vcpkg list`

     * 需執行 `vcpkg install drogon[ctl]`（32 位元）或 `vcpkg install drogon[ctl]:x64-windows`（64 位元）以包含 drogon_ctl。更多安裝選項請執行 `vcpkg search drogon` 查詢。

  2. 加入 ***drogon_ctl*** 指令與相依到環境變數 ***path***：

     ```
     C:\Dev\vcpkg\installed\x64-windows\tools\drogon
     C:\Dev\vcpkg\installed\x64-windows\bin
     C:\Dev\vcpkg\installed\x64-windows\lib
     C:\Dev\vcpkg\installed\x64-windows\include
     ```

  3. 重新啟動 ***powershell***，輸入 `drogon_ctl` 或 `drogon_ctl.exe`，若出現：

     ```
     usage: drogon_ctl [-v | --version] [-h | --help] <command> [<args>]
     commands list:
     create                  create some source files(Use 'drogon_ctl help create' for more information)
     help                    display this message
     press                   Do stress testing(Use 'drogon_ctl help press' for more information)
     version                 display version of this tool
     ```

     即表示安裝成功。

> 說明：
> 你需要熟悉用這些產生 CPP 函式庫的工具：
> `gcc` 或 `g++`（可用 [msys2](https://www.msys2.org/)、[mingw-w64](https://www.mingw-w64.org/)、[tdm-gcc](https://jmeubank.github.io/tdm-gcc/download/)）或 Microsoft Visual Studio 編譯器。

> 建議使用 **make.exe/nmake.exe/ninja.exe** 來建構，因為其配置與行為與 Linux 上的 make 一致，若使用 Linux/Windows 混合開發再發佈到 Linux，可減少錯誤。

* #### 使用 docker 映像檔

  我們也在 [docker hub](https://hub.docker.com/r/drogonframework/drogon) 提供了建構好的 docker 映像檔。在此 docker 內 Drogon 及其所有相依都已安裝完成，使用者可直接開發 Drogon 應用程式。

* #### 使用 Nix 套件

  Nix 套件管理器在 21.11 版後提供了 Drogon 的 Nix 套件。

  > 若尚未安裝 Nix，可依照 [NixOS 官網](https://nixos.org/download.html) 說明操作。

  你可以在專案根目錄加入以下 `shell.nix` 使用 Drogon 套件：

  ```nix
  { pkgs ? import <nixpkgs> {} }:
  pkgs.mkShell {
    nativeBuildInputs = with pkgs; [
      cmake
    ];

    buildInputs = with pkgs; [
      drogon
    ];
  }
  ```

  執行 `nix-shell` 進入 shell。會安裝 Drogon，並讓你擁有所有相依的環境。

  Drogon 的 Nix 套件有一些選項，可依需求設定：

  | 選項            | 預設值 |
  | --------------- | ------ |
  | sqliteSupport   | true   |
  | postgresSupport | false  |
  | redisSupport    | false  |
  | mysqlSupport    | false  |

  以下是如何更改選項值的範例：

  ```nix
    buildInputs = with pkgs; [
      (drogon.override {
        sqliteSupport = false;
      })
    ];
  ```

* #### 使用 CPM.cmake

  你可以用 [CPM.cmake](https://github.com/cpm-cmake/CPM.cmake) 包含 drogon 原始碼：

  ```cmake
  include(cmake/CPM.cmake)

  CPMAddPackage(
      NAME drogon
      VERSION 1.7.5
      GITHUB_REPOSITORY drogonframework/drogon
      GIT_TAG v1.7.5
  )

  target_link_libraries(${PROJECT_NAME} PRIVATE drogon)
  ```

* #### 直接包含 drogon 原始碼

  當然，你也可以在專案中包含 drogon 原始碼，例如將 drogon 放在專案目錄的 third_party 下，只需在專案的 cmake 檔案加入以下兩行：

  ```cmake
  add_subdirectory(third_party/drogon)
  target_link_libraries(${PROJECT_NAME} PRIVATE drogon)
  ```

# 下一步：[快速開始](/CHN/CHN-03-快速開始)

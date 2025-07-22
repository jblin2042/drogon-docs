[English](/ENG/ENG-08-1-Database-DbClient)

# 建立 DbClient

建立 DbClient 物件有兩種方式，一種是透過 DbClient 類的靜態方法（見 DbClient.h 標頭檔）：

```c++
#if USE_POSTGRESQL
    static std::shared_ptr<DbClient> newPgClient(const std::string &connInfo, const size_t connNum);
#endif
#if USE_MYSQL
    static std::shared_ptr<DbClient> newMysqlClient(const std::string &connInfo, const size_t connNum);
#endif
```

取得 DbClient 實作物件的智慧指標，connInfo 為連線字串，採 key=value 形式設定連線參數，詳情見標頭檔註解。connNum 為 DbClient 管理的連線數，影響併發度，請依實際需求設定。

此方式取得的物件，建議**持久化**（如放入全域容器），**不建議建立臨時物件用完即釋放**，原因如下：

* 浪費建立/斷開連線時間，增加系統延遲；
* 此介面為非阻塞，取得 DbClient 時其連線尚未建立，框架未提供連線成功回呼，若 sleep 等待再查詢違背非同步設計初衷。

因此，應在程式啟動時建立並持有 DbClient 物件。此工作可交由框架處理，故框架提供第二種建立方式：透過設定檔或 createDbClient 介面建立，設定方法見[設定檔](/CHN/CHN-10-配置文件#db_clients数据库客户端)。

需使用時，可透過框架介面取得 DbClient 智慧指標（需在 app.run() 之後呼叫）：

```c++
orm::DbClientPtr getDbClient(const std::string &name = "default");
```

name 為設定檔中的 name 欄位，用於區分同一應用的多個 DbClient。DbClient 管理的連線會自動斷線重連，使用者無需關心連線狀態，幾乎總是正常連線。

### 執行介面

DbClient 對外提供多種介面，包含：

```c++
/// 非同步介面
template <typename FUNCTION1,
          typename FUNCTION2,
          typename... Arguments>
void execSqlAsync(const std::string &sql,
                  FUNCTION1 &&rCallback,
                  FUNCTION2 &&exceptCallback,
                  Arguments &&... args) noexcept;

/// 非同步 future 介面
template <typename... Arguments>
std::future<const Result> execSqlAsyncFuture(const std::string &sql,
                                             Arguments &&... args) noexcept;

/// 同步介面
template <typename... Arguments>
const Result execSqlSync(const std::string &sql,
                         Arguments &&... args) noexcept(false);

/// 流式介面
internal::SqlBinder operator<<(const std::string &sql);
```

因可綁定任意數量與型別參數，這些介面皆為函式模板。

各介面性質如下表：

| 介面                                         | 同步/非同步 | 阻塞/非阻塞                     | 例外                                |
| :------------------------------------------* | :-------* | :--------------------------* | :--------------------------------* |
| void execSqlAsync                            | 非同步      | 非阻塞                        | 不丟出例外                             |
| std::future<const Result> execSqlAsyncFuture | 非同步      | 呼叫 future 的 get 方法時阻塞       | 呼叫 future 的 get 方法時可能丟出例外       |
| const Result execSqlSync                     | 同步      | 阻塞                          | 可能丟出例外                           |
| internal::SqlBinder operator<<               | 非同步      | 預設非阻塞，也可阻塞           | 不丟出例外                            |

同步介面通常涉及網路 IO 皆為阻塞，非同步介面則為非阻塞，但非同步介面也可阻塞（即等回呼執行完才返回）。DbClient 非同步介面阻塞時，回呼在同一執行緒執行，介面才結束。

高併發場景請選用非同步非阻塞介面，低併發（如設備管理頁）可選同步介面。

* #### execSqlAsync

  ```c++
  template <typename FUNCTION1,
          typename FUNCTION2,
          typename... Arguments>
  void execSqlAsync(const std::string &sql,
                  FUNCTION1 &&rCallback,
                  FUNCTION2 &&exceptCallback,
                  Arguments &&... args) noexcept;
  ```

  最常用的非同步介面，預設非阻塞。

  sql 為 SQL 字串，若需綁定參數，依資料庫占位符規則（PostgreSQL 用 $1, $2...，MySQL 用 ?）。

  不定參數 args 為綁定參數，數量與型別需與 SQL 占位符一致，支援：

  * 整數型別：各種字長，應與資料庫欄位型別相符；
  * 浮點型別：`float` 或 `double`，應與資料庫欄位型別相符；
  * 字串型別：`std::string` 或 `const char[]`，對應資料庫字串型別或可用字串表示型別；
  * 日期型別：`trantor::Date`，對應資料庫 date、datetime、timestamp 等型別；
  * 二進位型別：`std::vector<char>`，對應 PostgreSQL 的 bytea 或 MySQL 的 blob 型別；

  參數可為左值、右值、變數或常量，皆可。

  rCallback 與 exceptCallback 分別為結果回呼與例外回呼，定義如下：

  * 結果回呼：void (const Result &)，可傳入 std::function、lambda 等；
  * 例外回呼：void (const DrogonDbException &)，可傳入相符型別可呼叫物件；

  SQL 執行成功，結果由 Result 類包裝並透過結果回呼傳回；若有例外，則執行例外回呼，可從 DrogonDbException 取得例外資訊。

  範例：

  ```c++
  auto clientPtr = drogon::app().getDbClient();
  clientPtr->execSqlAsync("select * from users where org_name=$1",
                              [](const drogon::orm::Result &result) {
                                  std::cout << r.size() << " rows selected!" << std::endl;
                                  int i = 0;
                                  for (auto row : result)
                                  {
                                      std::cout << i++ << ": user name is " << row["user_name"].as<std::string>() << std::endl;
                                  }
                              },
                              [](const DrogonDbException &e) {
                                  std::cerr << "error:" << e.base().what() << std::endl;
                              },
                              "default");
  ```

  Result 物件為標準容器，支援迭代器，可用範圍迴圈取每一行。Result、Row、Field 物件介面詳見原始碼。

  DrogonDbException 為所有資料庫例外基類，詳情見原始碼註解。

* #### execSqlAsyncFuture

  ```c++
  template <typename... Arguments>
  std::future<const Result> execSqlAsyncFuture(const std::string &sql,
                                              Arguments &&... args) noexcept;
  ```

  非同步 future 介面，省略前介面的回呼參數，回傳 future 物件，需呼叫 get() 取得結果，例外需用 try/catch 捕獲，否則 SQL 執行例外時程式會終止。

  範例：

  ```c++
  auto f = clientPtr->execSqlAsyncFuture("select * from users where org_name=$1",
                                      "default");
  try
  {
      auto result = f.get(); // 阻塞直到取得結果或捕獲例外
      std::cout << result.size() << " rows selected!" << std::endl;
      int i = 0;
      for (auto row : result)
      {
          std::cout << i++ << ": user name is " << row["user_name"].as<std::string>() << std::endl;
      }
  }
  catch (const DrogonDbException &e)
  {
      std::cerr << "error:" << e.base().what() << std::endl;
  }
  ```

* #### execSqlSync

  ```c++
  template <typename... Arguments>
  const Result execSqlSync(const std::string &sql,
                          Arguments &&... args) noexcept(false);
  ```

  同步介面最簡單直觀，輸入 SQL 字串與綁定參數，回傳 Result 物件，呼叫時會阻塞執行緒，若錯誤則丟出例外，需用 try/catch 捕獲。

  範例：

  ```c++
  try
  {
      auto result = clientPtr->execSqlSync("update users set user_name=$1 where user_id=$2",
                                          "test",
                                          1); // 阻塞直到取得結果或捕獲例外
      std::cout << result.affectedRows() << " rows updated!" << std::endl;
  }
  catch (const DrogonDbException &e)
  {
      std::cerr << "error:" << e.base().what() << std::endl;
  }
  ```

* #### operator<<

  ```c++
  internal::SqlBinder operator<<(const std::string &sql);
  ```

  流式介面較特殊，SQL 與參數依序用 `<<` 輸入，`>>` 指定結果回呼與例外回呼。前述 select 範例用流式介面如下：

  ```c++
  *clientPtr  << "select * from users where org_name=$1"
              << "default"
              >> [](const drogon::orm::Result &result)
                  {
                      std::cout << result.size() << " rows selected!" << std::endl;
                      int i = 0;
                      for (auto row : result)
                      {
                          std::cout << i++ << ": user name is " << row["user_name"].as<std::string>() << std::endl;
                      }
                  }
              >> [](const DrogonDbException &e)
                  {
                      std::cerr << "error:" << e.base().what() << std::endl;
                  };
  ```

  此寫法與第一種非同步非阻塞介面完全等效，採用哪種介面依使用習慣。若要阻塞模式，可用 `<<` 輸入 `Mode::Blocking` 參數。

  流式介面還有特殊用法，可用特殊結果回呼讓框架逐行傳回結果，回呼型別如下：

  ```c++
  void (bool,Arguments...);
  ```

  第一個 bool 參數為 true 表示本次回呼為空行，即所有結果已回傳，為最後一次回呼；
  後面為一系列參數，對應一行記錄每一欄的值，框架會自動型別轉換，使用者需注意型別匹配。可用 const 左值引用、右值引用或值型別。

  例子重寫如下：

  ```c++
  int i = 0;
  *clientPtr  << "select user_name, user_id from users where org_name=$1"
              << "default"
              >> [&i](bool isNull, const std::string &name, int64_t id)
                      {
                      if (!isNull)
                          std::cout << i++ << ": user name is " << name << ", user id is " << id << std::endl;
                      else
                          std::cout << i << " rows selected!" << std::endl;
                      }
              >> [](const DrogonDbException &e)
                  {
                      std::cerr << "error:" << e.base().what() << std::endl;
                  };
  ```

  可見 select 語句中的 user_name 和 user_id 欄位值分別賦給回呼中的 name 與 id 變數，無需自行處理型別轉換，使用上更便利。

> **注意：此例中的變數 i，使用者必須確保回呼發生時 i 仍有效，因為是引用捕獲，回呼可能在其他執行緒執行，當下文環境可能已失效。常見做法是用智慧指標持有臨時變數，再由回呼捕獲，確保變數有效性。**

### 總結

每個 DbClient 物件僅有一個自己的 EventLoop 執行緒，負責控制資料庫連線 IO，透過非同步或同步介面接收請求，再用回呼函式回傳結果。

雖然也提供阻塞介面，但僅阻塞呼叫者執行緒，只要呼叫者不是 EventLoop 執行緒，不會影響 EventLoop 正常運作。回呼執行時，程式運行於 EventLoop 執行緒，故勿在回呼內執行任何阻塞操作，否則會影響資料庫併發，熟悉 non-blocking I/O 編程者應明白此限制。

# 下一步：[交易](/CHN/CHN-08-2-数据库-事务)

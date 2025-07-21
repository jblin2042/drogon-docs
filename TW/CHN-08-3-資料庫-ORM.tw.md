[English](/ENG/ENG-08-3-Database-ORM)

# 資料庫 - ORM

## Model

使用 Drogon 的 ORM 功能，首先需建立 Model 類。Drogon 的命令列工具 `drogon_ctl` 可自動根據指定資料庫的表結構產生 Model 類原始碼。使用時只需 include 對應標頭檔。

每個 Model 類對應一個資料表，每個 Model 實例對應表中的一筆資料。

建立 Model 類命令如下：

```shell
drogon_ctl create model <model_path>
```

最後一個參數為 Model 存放路徑，該路徑需有 model.json 設定檔，內容如下：

```json
{
  "rdbms": "postgresql",
  "host": "127.0.0.1",
  "port": 5432,
  "dbname": "test",
  "user": "test",
  "passwd": "",
  "tables": [],
  "relationships": {
      "enabled": false,
      "items": []
  }
}
```

設定參數與應用設定檔一致，詳見[設定檔](/CHN/CHN-10-配置文件#db_clients)。

tables 為字串陣列，指定要產生 Model 的表名，若空則所有表都會產生 Model 類。

用 drogon_ctl create project 建立的專案已預設 models 目錄及 model.json，編輯後即可產生 Model 類。

## Model 類介面

主要有 getter 與 setter 兩類：

getter 分為：

* getColumnName：取得欄位的智慧指標，方便判斷是否為 NULL。
* getValueOfColumnName：取得欄位值，回傳常引用，若為 NULL 則回傳預設值。

二進位欄位（blob, bytea）有 getValueOfColumnNameAsString，將資料裝入 std::string。

setter 用於設定欄位值，型別與欄位對應。自動產生欄位（如自增主鍵）無 setter。

toJson() 可將 Model 轉為 JSON，二進位欄位採 base64 編碼。

Model 類的靜態成員可取得表資訊，如 Cols 可獲得所有欄位名稱，方便自動提示。

## Mapper 類模板

Model 與資料表的映射由 Mapper 類模板負責，封裝常見增刪改查操作，無需撰寫 SQL。

Mapper 建構時指定 Model 類型，建構參數為 DbClient 智慧指標。Transaction 亦可作為 DbClient 使用，故 Mapper 也支援交易。

Mapper 提供同步與非同步介面。同步介面阻塞且可能丟出例外，future 物件 get() 時阻塞且可能丟例外。非同步介面透過結果回呼與例外回呼回傳結果，例外回呼型別與 DbClient 一致，結果回呼依功能不同分多種。詳見下列圖示：

![](images/mapper_method1.png)
![](images/mapper_method2.png)
![](images/mapper_method3.png)

> **注意：使用交易時，例外不必然導致回滾。若 findByPrimaryKey 未找到資料、findOne 找到多於或少於一筆資料，會丟例外或進入例外回呼，例外型別為 UnexpectedRows。若需回滾，請顯式呼叫 rollback()。**

## 條件物件

多數介面需傳入條件物件，即 Criteria 類，表示 where 條件（如大於、等於、小於、isNull 等）。

```c++
template <typename T>
Criteria(const std::string &colName, const CompareOperator &opera, T &&arg)
```

第一參數為欄位名，第二為比較型態，第三為比較值。若型態為 IsNull/IsNotNull，則不需第三參數。

例如：

```c++
Criteria("user_id",CompareOperator::EQ,1);
```

可寫成：

```c++
Criteria(Users::Cols::_user_id,CompareOperator::EQ,1);
```

此寫法利於自動提示且不易出錯。

Criteria 也支援自訂 where 條件：

```c++
template <typename... Arguments>
explicit Criteria(const CustomSql &sql, Arguments &&...args)
```

第一參數為含 `$?` 占位符的 CustomSql，第二為綁定參數，行為同 [execSqlAsync](/CHN/CHN-08-1-数据库-Dbclient.md#execsqlasync)。

例如：

```c++
Criteria(CustomSql("tags @> $?"), "cloud");
```

或：

```c++
Criteria("tags @> $?"_sql, "cloud");
```

條件物件支援 AND/OR 運算，可方便組合複雜 where 條件：

```c++
Mapper<Users> mp(dbClientPtr);
auto users = mp.findBy(
(Criteria(Users::Cols::_user_name,CompareOperator::Like,"李%")&&Criteria(Users::Cols::_gender,CompareOperator::EQ,0))
||(Criteria(Users::Cols::_user_name,CompareOperator::Like,"王%")&&Criteria(Users::Cols::_gender,CompareOperator::EQ,1))
));
```

查詢 users 表所有姓李的男士或姓王的女士。

## Mapper 鏈式介面

常見 SQL 約束（limit, offset 等）Mapper 也支援，採鏈式介面，可串接多個約束。執行任一操作後約束即清空，僅本次操作有效：

```c++
Mapper<Users> mp(dbClientPtr);
auto users = mp.orderBy(Users::Cols::_join_time).limit(25).offset(0).findAll();
```

查詢 users 表，分頁每頁 25 筆，取第一頁。

詳見 Mapper.h。

## 轉換

`convert` 設定為模型專用。讀寫資料庫前後可加轉換層。`enabled` 為布林值，`items` 為物件陣列，包含：

* `table`: 需轉換欄位的表名
* `column`: 欄位名
* `method`: 轉換方法物件
  * `after_db_read`: 讀出後呼叫的方法，簽名：void([const] std::shared_ptr<type> [&])
  * `before_db_write`: 寫入前呼叫的方法，簽名同上
* `includes`: 需 include 的檔案字串陣列

## 關聯

資料表間關聯可於 model.json 設定 relationships。採手動設定，因實務上常不使用外鍵。

`enable` 為 true 時，Model 會依設定產生對應介面。

關聯分三種：has one、has many、many to many。

* ### has one

  1 對 1 關聯，例：產品表與庫存單位表。設定如下：

  ```json
  {
    "type": "has one",
    "original_table_name": "products",
    "original_table_alias": "product",
    "original_key": "id",
    "target_table_name": "skus",
    "target_table_alias": "SKU",
    "target_key": "product_id",
    "enable_reverse": true
  }
  ```

  依此設定，products Model 會有：

  ```c++
      void getSKU(const DbClientPtr &clientPtr,
                  const std::function<void(Skus)> &rcb,
                  const ExceptionCallback &ecb) const;
  ```

  若 enable_reverse 為 true，skus Model 會有：

  ```c++
      void getProduct(const DbClientPtr &clientPtr,
                      const std::function<void(Products)> &rcb,
                      const ExceptionCallback &ecb) const;
  ```

* ### has many

  一對多關聯，例：產品與評價。設定如下：

  ```json
  {
    "type": "has many",
    "original_table_name": "products",
    "original_table_alias": "product",
    "original_key": "id",
    "target_table_name": "reviews",
    "target_table_alias": "",
    "target_key": "product_id",
    "enable_reverse": true
  }
  ```

  products Model 會有：

  ```c++
      void getReviews(const DbClientPtr &clientPtr,
                      const std::function<void(std::vector<Reviews>)> &rcb,
                      const ExceptionCallback &ecb) const;
  ```

  reviews Model 會有：

  ```c++
      void getProduct(const DbClientPtr &clientPtr,
                      const std::function<void(Products)> &rcb,
                      const ExceptionCallback &ecb) const;
  ```

* ### many to many

  多對多關聯，例：產品與購物車。設定如下：

  ```json
  {
    "type": "many to many",
    "original_table_name": "products",
    "original_table_alias": "",
    "original_key": "id",
    "pivot_table": {
      "table_name": "carts_products",
      "original_key": "product_id",
      "target_key": "cart_id"
    },
    "target_table_name": "carts",
    "target_table_alias": "",
    "target_key": "id",
    "enable_reverse": true
  }
  ```

  products Model 會有：

  ```c++
      void getCarts(const DbClientPtr &clientPtr,
                    const std::function<void(std::vector<std::pair<Carts,CartsProducts>>)> &rcb,
                    const ExceptionCallback &ecb) const;
  ```

  carts Model 會有：

  ```c++
      void getProducts(const DbClientPtr &clientPtr,
                      const std::function<void(std::vector<std::pair<Products,CartsProducts>>)> &rcb,
                      const ExceptionCallback &ecb) const;
  ```

## Restful API 控制器

drogon_ctl 可於建立 Model 同時產生 RESTful 風格 controller，讓使用者零程式碼即可對表進行 CRUD 操作。API 支援主鍵查詢、條件查詢、排序、指定欄位、欄位別名等。由 model.json 的 restful_api_controllers 設定控制，詳見 json 註解。

每個表的 controller 由基類與子類組成，基類與表結構密切相關，子類供使用者擴充業務邏輯或修飾介面。此設計可在表結構變更時只更新基類、不覆蓋子類，確保開發連續性。

此功能非普遍需求，故不詳述。

# 下一步：[FastDbClient](/CHN/CHN-08-4-数据库-FastDbClient)

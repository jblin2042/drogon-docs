[English](/ENG/ENG-18-Redis) 

# Redis

Drogon 支援 Redis，Redis 是極高速的記憶體資料儲存，可作為資料庫快取或訊息代理。與 Drogon 其他元件一樣，Redis 操作皆為非同步，確保 Drogon 在高負載下仍能高併發運作。

Redis 需依賴 hiredis 函式庫。若建置 Drogon 時未安裝 hiredis，則無法使用 Redis 功能。

## 建立客戶端

Redis 客戶端可如下建立：

```c++
app().createRedisClient("127.0.0.1", 6379);
// ...
// 在 app.run() 之後
RedisClientPtr redisClient = app().getRedisClient();
```

同資料庫客戶端，Redis 亦支援 config 檔設定，亦可設為 Fast 模式，設定如下：

```json
    "redis_clients": [
        {
            //name: 客戶端名稱, 預設 'default'
            //"name":"",
            //host: 伺服器 IP, 預設 127.0.0.1
            "host": "127.0.0.1",
            //port: 伺服器埠號, 預設 6379
            "port": 6379,
            //passwd: 密碼，預設空
            "passwd": "",
            //db index: 預設 0
            "db": 0,
            //is_fast: 預設 false, 是否為 fast 模式，true 時僅能於 IO 執行緒或主執行緒使用，且不可用同步介面
            "is_fast": false,
            //number_of_connections: 連線數, 預設 1, is_fast 為 true 時表示每個 IO 執行緒或主執行緒的連線數，否則為全部連線數
            "number_of_connections": 1,
            //timeout: 逾時值，預設 -1.0，單位秒，表示命令逾時，超過即回傳逾時錯誤，0 或負值表示無逾時限制
            "timeout": -1.0
        }
    ]
```

## 使用 Redis

`execCommandAsync` 以非同步方式執行 Redis 指令。至少需三個參數，前兩個為指令成功或失敗時的回呼，第三為指令本身，可用 C 風格格式字串，其餘為格式字串參數。例如設置 `name` 為 `drogon`：

```c++
redisClient->execCommandAsync(
    [](const drogon::nosql::RedisResult &r) {},
    [](const std::exception &err) {
        LOG_ERROR << "something failed!!! " << err.what();
    },
    "set name drogon");
```

或設置 `myid` 為 `587d-4709-86e4`：

```c++
redisClient->execCommandAsync(
    [](const drogon::nosql::RedisResult &r) {},
    [](const std::exception &err) {
        LOG_ERROR << "something failed!!! " << err.what();
    },
    "set myid %s", "587d-4709-86e4");
```

同理可用 `execCommandAsync` 取得資料：

```c++
redisClient->execCommandAsync(
    [](const drogon::nosql::RedisResult &r) {
        if (r.type() == RedisResultType::kNil)
            LOG_INFO << "Cannot find variable associated with the key 'name'";
        else
            LOG_INFO << "Name is " << r.asString();
    },
    [](const std::exception &err) {
        LOG_ERROR << "something failed!!! " << err.what();
    },
    "get name");
```

## Redis 交易

Redis 交易允許一次執行多個指令。交易內所有指令依序執行，其他客戶端指令不會插入交易**中間**。注意 Redis 交易非原子操作，收到 EXEC 指令後進入交易，交易中任一指令失敗，其餘指令仍會執行，無回滾。

`newTransactionAsync` 建立新交易，之後可如一般 RedisClient 使用，最後以 `RedisTransaction::execute` 執行交易。

```c++
redisClient->newTransactionAsync([](const RedisTransactionPtr &transPtr) {
    transPtr->execCommandAsync(
        [](const drogon::nosql::RedisResult &r) { /* this command works */ },
        [](const std::exception &err) { /* this command failed */ },
    "set name drogon");

    transPtr->execute(
        [](const drogon::nosql::RedisResult &r) { /* transaction worked */ },
        [](const std::exception &err) { /* transaction failed */ });
});
```

## 協程

Redis 客戶端亦支援協程。需 GCC 11 或更新編譯器，並以 `cmake -DCMAKE_CXX_FLAGS="-std=c++20"` 啟用。詳見[協程](/CHN/CHN-16-协程)。

```c++
try
{
    auto transaction = co_await redisClient->newTransactionCoro();
    co_await transaction->execCommandCoro("set zzz 123");
    co_await transaction->execCommandCoro("set mening 42");
    co_await transaction->executeCoro();
}
catch(const std::exception& e)
{
    LOG_ERROR << "Redis failed: " << e.what();
}
```

# 下一步：[測試框架](/CHN/CHN-19-测试框架)

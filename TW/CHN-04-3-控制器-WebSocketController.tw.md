[English](/ENG/ENG-04-3-Controller-WebSocketController)

# WebSocketController

顧名思義，`WebSocketController` 用於處理 WebSocket 邏輯。WebSocket 是基於 HTTP 的長連線方案，建立初期會有一次 HTTP 格式的請求與回應交換，建立完成後，所有訊息都在 WebSocket 上傳輸，訊息有固定格式包裝，但內容與收發順序完全由使用者自訂。

### 產生

可用 `drogon_ctl` 工具快速產生 `WebSocketController` 原始檔，指令格式如下：

```bash
drogon_ctl create controller -w <[namespace::]class_name>
```

假設要用 WebSocket 實作簡單的 Echo 功能（伺服器將收到的訊息原封不動回傳），可用 `drogon_ctl` 建立 `WebSocketController` 實作類 EchoWebsock：

```bash
drogon_ctl create controller -w EchoWebsock
```

此指令會產生 EchoWebsock.h 和 EchoWebsock.cc 兩個檔案：

```c++
// EchoWebsock.h
#pragma once
#include <drogon/WebSocketController.h>
using namespace drogon;
class EchoWebsock:public drogon::WebSocketController<EchoWebsock>
{
  public:
    void handleNewMessage(const WebSocketConnectionPtr&,
                          std::string &&,
                          const WebSocketMessageType &) override;
    void handleNewConnection(const HttpRequestPtr &,
                             const WebSocketConnectionPtr&) override;
    void handleConnectionClosed(const WebSocketConnectionPtr&) override;
    WS_PATH_LIST_BEGIN
    // 在此定義路徑
    WS_PATH_LIST_END
};
```

```c++
// EchoWebsock.cc
#include "EchoWebsock.h"
void EchoWebsock::handleNewMessage(const WebSocketConnectionPtr &wsConnPtr,std::string &&message)
{
    // 在此撰寫應用邏輯
}
void EchoWebsock::handleNewConnection(const HttpRequestPtr &req,const WebSocketConnectionPtr &wsConnPtr)
{
    // 在此撰寫應用邏輯
}
void EchoWebsock::handleConnectionClosed(const WebSocketConnectionPtr &wsConnPtr)
{
    // 在此撰寫應用邏輯
}
```

### 使用方式

* #### 路徑映射

  編輯後內容如下：

  ```c++
  // EchoWebsock.h
  #pragma once
  #include <drogon/WebSocketController.h>
  using namespace drogon;
  class EchoWebsock:public drogon::WebSocketController<EchoWebsock>
  {
  public:
      virtual void handleNewMessage(const WebSocketConnectionPtr&,
                                  std::string &&,
                                  const WebSocketMessageType &)override;
      virtual void handleNewConnection(const HttpRequestPtr &,
                                      const WebSocketConnectionPtr&)override;
      virtual void handleConnectionClosed(const WebSocketConnectionPtr&)override;
      WS_PATH_LIST_BEGIN
      // 在此定義路徑
      WS_PATH_ADD("/echo");
      WS_PATH_LIST_END
  };
  ```

  ```c++
  // EchoWebsock.cc
  #include "EchoWebsock.h"
  void EchoWebsock::handleNewMessage(const WebSocketConnectionPtr &wsConnPtr,std::string &&message)
  {
      // 在此撰寫應用邏輯
      wsConnPtr->send(message);
  }
  void EchoWebsock::handleNewConnection(const HttpRequestPtr &req,const WebSocketConnectionPtr &wsConnPtr)
  {
      // 在此撰寫應用邏輯
  }
  void EchoWebsock::handleConnectionClosed(const WebSocketConnectionPtr &wsConnPtr)
  {
      // 在此撰寫應用邏輯
  }
  ```

  在此範例中，透過 `WS_PATH_ADD` 巨集將控制器註冊到 `/echo` 路徑，`WS_PATH_ADD` 用法與其他控制器巨集類似，也可註冊路徑並附加 [中介軟體與過濾器](/CHN/CHN-05-中间件和过滤器)。WebSocket 在框架中獨立處理，因此可與前兩種控制器路徑重複而不互相影響。

  此範例三個虛擬函式的實作，只有 handleNewMessage 有實質內容，就是將收到的訊息用 send 介面回傳給客戶端。將此控制器編譯進框架即可看到效果，請自行試驗。

  > **注意：如同一般 HTTP 協定，WebSocket 也可被旁路還原，若需安全保障，建議用 HTTPS 提供加密功能。當然，使用者也可自行在伺服器與客戶端加密/解密，只是 HTTPS 更方便，底層由 drogon 處理，使用者只需專注業務邏輯。**

  使用者自訂的 `WebSocketController` 類別需繼承 `drogon::WebSocketController` 類模板，模板參數為子類型，並需實作以下三個虛擬函式以處理 WebSocket 建立、關閉與訊息：

  ```c++
  virtual void handleNewConnection(const HttpRequestPtr &req,const WebSocketConnectionPtr &wsConn);
  virtual void handleNewMessage(const WebSocketConnectionPtr &wsConn,std::string &&message,
  const WebSocketMessageType &);
  virtual void handleConnectionClosed(const WebSocketConnectionPtr &wsConn);
  ```

  說明如下：

  * handleNewConnection：WebSocket 建立後呼叫，req 為客戶端送來的建立請求，此時框架已回應，使用者可透過 req 取得額外資訊（如 token）。wsConn 為 WebSocket 物件智慧指標，常用介面後述。
  * handleNewMessage：WebSocket 收到新訊息後呼叫，訊息存於 message 變數，message 為完整訊息內容，框架已完成解封包與解碼，使用者直接處理即可。
  * handleConnectionClosed：WebSocket 連線關閉後呼叫，可做收尾工作。

* #### 介面

  WebSocketConnection 物件常用介面如下：

  ```c++
  // 傳送 WebSocket 訊息，編碼與封包由框架處理，直接傳送訊息內容
  void send(const char *msg,uint64_t len);
  void send(const std::string &msg);

  // 本 WebSocket 的本端與遠端位址
  const trantor::InetAddress &localAddr() const;
  const trantor::InetAddress &peerAddr() const;

  // 本 WebSocket 的連線狀態
  bool connected() const;
  bool disconnected() const;

  // 關閉本 WebSocket
  void shutdown();//close write
  void forceClose();//close

  // 設定與取得本 WebSocket 的上下文，可存放業務資料，
  // any 型別可存取任意型別物件。
  void setContext(const any &context);
  const any &getContext() const;
  any *getMutableContext();
  ```

# 下一步：[中介軟體與過濾器](/CHN/CHN-05-中间件和过滤器)

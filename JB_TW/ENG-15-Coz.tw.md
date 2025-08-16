## 使用 Coz 進行因果分析（Causal profiling）

[原文：ENG-15-Coz.md](/ENG/ENG-15-Coz.md)

Coz 可分析兩項指標：

* 吞吐量（throughput）
* 延遲（latency）

若要分析應用程式吞吐量，請於 cmake 啟用 `COZ_PROFILING` 選項，並以 `Debug` 或 `RelWithDebInfo` 模式編譯，確保可執行檔含除錯資訊。如此會於處理請求時插入 coz 進度點。全域延遲分析目前尚未支援，但可於使用者程式碼中進行。

編譯完成後，需以 coz profiler 執行可執行檔，例如：

```shell
coz run --- [執行檔路徑]
```

最後，應對應用程式進行壓力測試，建議涵蓋所有程式路徑並執行足夠時間（15 分鐘以上）。

分析結果會產生 `profile.coz` 檔於目前目錄。可用官方 [viewer](https://plasma-umass.org/coz/) 開啟，或自官方 [git repo](https://github.com/plasma-umass/coz) 下載本地版。

Coz 亦支援以 `--source-scope <pattern>` 或 `-s <pattern>` 限定分析檔案範圍，十分實用。

更多資訊請參考：

* `coz run --help`
* [Git repo](https://github.com/plasma-umass/coz)
* [Coz 白皮書](https://arxiv.org/pdf/1608.03676v1.pdf)

## 下一步: [Brotli 壓縮](/JB_TW/ENG-16-Brotli.tw.md)

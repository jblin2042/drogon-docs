[English](/ENG/ENG-15-Coz)

# 分析 - Coz

使用 Coz 可分析兩項指標：

* 吞吐量
* 延遲

若要分析應用程式吞吐量，請開啟 `COZ_PROFILING` cmake 選項，並於可執行檔中以 cmake 的 `Debug` 或 `RelWithDebInfo` 模式編譯，包含除錯資訊。如此可於處理請求時加入 Coz 進度點。目前全域延遲分析尚未支援，但可於使用者程式碼中進行。

編譯含進度點的應用程式後，需用 Coz 分析器執行，例如 `coz run --- [可執行檔路徑]`。

最後，應對程式進行壓力測試，為獲最佳結果，請壓測所有程式路徑並執行長時間分析（建議 15 分鐘以上）。

分析結束後，會於目前工作目錄產生 `profile.coz` 檔案。可於官方 [viewer](https://plasma-umass.org/coz/) 開啟分析檔，或自官方 [git repo](https://github.com/plasma-umass/coz) 下載本地檢視。

Coz 亦支援 `--source-scope <pattern>` 或 `-s <pattern>` 等選項，限定分析檔中包含的原始檔範圍，十分實用。

更多資訊請參考：

- `coz run --help`
- [Git repo](https://github.com/plasma-umass/coz)
- [Coz whitepaper](https://arxiv.org/pdf/1608.03676v1.pdf)

# 下一步：[Brotli 壓縮](/CHN/CHN-16-Brotli压缩)

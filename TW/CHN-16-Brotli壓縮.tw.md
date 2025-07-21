[English](/ENG/ENG-16-Brotli)

# 壓縮 - Brotli

Drogon 原生支援 Brotli 靜態壓縮檔案，只要存在對應的 Brotli 壓縮檔即可。

例如，請求 `/path/to/asset.js` 時，Drogon 會自動搜尋 `/path/to/asset.js.br`。

此功能預設於 `config.json` 中將 `br_static` 設為 `true`。

若需啟用 Brotli 動態壓縮，請於 `config.json` 將 `use_brotli` 設為 `true`。

若不使用 Brotli 靜態壓縮，可於 `config.json` 將 `br_static` 設為 `false`，以避免額外的同名檔案檢查。

# 下一步：[協程](/CHN/CHN-17-协程)

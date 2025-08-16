## Brotli 壓縮說明

[原文：ENG-16-Brotli.md](/ENG/ENG-16-Brotli.md)

Drogon 原生支援 Brotli 靜態壓縮檔案，只要資源旁有對應的 Brotli 壓縮檔即可。

例如，若請求 `/path/to/asset.js`，Drogon 會自動尋找 `/path/to/asset.js.br`。

此功能預設於 `config.json` 中將 `br_static` 設為 `true`。

若需動態以 Brotli 壓縮，請於 `config.json` 設定 `use_brotli` 為 `true`。

若不需 Brotli 靜態壓縮，可將 `br_static` 設為 `false`，避免多餘的檔案檢查。

## 下一步: [協程](/JB_TW/ENG-17-Coroutines.tw.md)

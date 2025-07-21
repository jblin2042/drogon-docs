[English](/ENG/ENG-10-Plugins)

# 插件

插件可協助使用者建構複雜應用程式，在 Drogon 中，所有插件皆由框架依據設定檔統一建立並安裝至應用。Drogon 的插件皆為單一實例，使用者可用插件實作任何所需功能。

Drogon 執行 run() 介面時，會依設定檔逐一實體化每個插件並呼叫其 `initAndStart()` 介面。

## 設定

插件設定皆於設定檔完成，例如：

```json
    "plugins": [
        {
        //name: 插件類別名稱
        "name": "DataDictionary",
        //dependencies: 插件依賴的其他插件，可省略
        "dependencies": [],
        //config: 插件初始化設定，為傳入插件的 json 物件，可省略
        "config": {
        }
        }],
```

每個插件設定有三項：

* name：插件類別名稱（含命名空間），框架依此建立插件實例。註解此項則插件停用。
* dependencies：插件依賴的其他插件名稱列表，框架依序建立與初始化。被依賴插件先建立、先初始化，程式結束時反向順序關閉與銷毀。請勿產生循環依賴，Drogon 會檢查並報錯退出。註解此項則依賴列表為空。
* config：初始化插件的 json 物件，使用者可自訂任何設定，該物件會傳入插件的 `initAndStart()` 介面。註解此項則傳入空物件。

## 定義

自訂插件必須繼承 drogon::Plugin 類模板，模板參數即插件型別，例如：

```c++
class DataDictionary : public drogon::Plugin<DataDictionary>
{
public:
    virtual void initAndStart(const Json::Value &config) override;
    virtual void shutdown() override;
    ...
};
```

命令列工具 drogon_ctl 也提供建立插件命令：

```shell
drogon_ctl create plugin <[namespace::]class_name>
```

可用此命令建立插件原始碼，再進一步編輯。

## 取得實例

插件實例由 drogon 建立，使用者可透過以下介面取得：

```c++
template <typename T> T *getPlugin();
```

或

```c++
PluginBase *getPlugin(const std::string &name);
```

第一種方式較方便，例如取得 DataDictionary 插件：

```c++
auto *pluginPtr=app().getPlugin<DataDictionary>();
```

注意，取得插件建議在框架 run() 介面呼叫後，否則可能取得未初始化的插件（只要使用時已初始化即可）。由於插件依賴順序初始化，只要依賴設定正確，在 `initAndStart()` 介面中取得其他插件實例是沒問題的。

## 生命週期

所有插件於 run() 介面內初始化完畢，應用程式結束時才銷毀，故插件生命週期幾乎等同應用程式，getPlugin() 介面不需回傳智慧指標。

# 下一步：[設定檔](/CHN/CHN-11-配置文件)

# .NET Framework 4.6.1 → 4.8 升級技術文件

**環境：** Windows Server 2016  
**升級目標：** .NET Framework 4.6.1 → 4.8  
**文件日期：** 2026-05-26  

---

## 1. 環境確認

| 項目 | 說明 | 狀態 |
|------|------|------|
| 作業系統 | Windows Server 2016 | ✅ |
| .NET Framework Runtime | 4.8 已安裝 | ✅ |
| CLR 版本 | v4.0（4.x 系列共用） | ✅ |

> .NET Framework 4.8 向下相容 4.6.1，主機環境條件足夠，無需額外安裝 Runtime。

---

## 2. 設定檔變更項目

### 2.1 `web.config`

#### 變更前
```xml
<compilation debug="true" strict="false" explicit="true" targetFramework="4.6.1"/>
<httpRuntime targetFramework="4.6.1"/>
```

#### 變更後
```xml
<compilation debug="true" strict="false" explicit="true" targetFramework="4.8"/>
<httpRuntime targetFramework="4.8"/>
```

#### 說明

| 設定 | 說明 |
|------|------|
| `compilation targetFramework` | 告訴編譯器以 4.8 方式編譯應用程式 |
| `httpRuntime targetFramework` | 控制 ASP.NET 執行期行為對應的版本語意，影響 encoding、routing、錯誤處理等預設值 |
| `debug="false"` | **正式環境必須設為 false**，避免效能損耗與安全風險 |
| `strict` / `explicit` | VB.NET 專用選項，C# 專案無影響 |

> ⚠️ `compilation` 與 `httpRuntime` 兩者都必須設定，缺一不可。  
> 只改 `compilation` 不會完整套用 4.8 的 runtime 行為。

---

### 2.2 `app.config`（若有桌面/Console 應用程式）

#### 變更前
```xml
<startup>
  <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.6.1" />
</startup>
```

#### 變更後
```xml
<startup>
  <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.8" />
</startup>
```

#### 說明

| 屬性 | 說明 |
|------|------|
| `version="v4.0"` | CLR 版本，.NET 4.x 全系列均為 `v4.0`，**不需修改** |
| `sku` | 實際要求的 .NET Framework 版本，改為 `v4.8` |

> 程式啟動時 CLR 會檢查主機是否符合此版本需求。  
> 建議與 `web.config` 的 `targetFramework` 保持一致，避免版本號混淆。

---

## 3. 升級後行為差異注意事項

### 3.1 TLS 預設版本變更
- .NET 4.7 起，TLS 預設改為**跟隨系統設定**（通常為 TLS 1.2）
- 若應用程式有對外 HTTPS 呼叫，請確認遠端服務支援 TLS 1.2
- 若需強制指定，可在程式啟動時加入：
```csharp
System.Net.ServicePointManager.SecurityProtocol = 
    System.Net.SecurityProtocolType.Tls12;
```

### 3.2 ASP.NET Runtime 行為變更
- `httpRuntime targetFramework` 從 4.6 改為 4.8 後，以下行為可能受影響：
  - `HttpRequest` 的 Encoding 處理方式
  - `MachineKey` 相關行為
  - URL Routing 細節

### 3.3 `AppContext` Switches
- 4.7 / 4.8 引入許多 `AppContext` 開關來控制新舊行為
- 若升級後發現異常行為，可透過 `app.config` 還原舊有行為：
```xml
<runtime>
  <AppContextSwitchOverrides value="Switch.SomeSwitch=true" />
</runtime>
```

### 3.4 `System.Uri` 解析行為
- 4.7+ 對部分 URI 解析有細微變更
- 若應用程式有複雜的 URL 處理邏輯，需特別測試

### 3.5 WebForms（ASP.NET）升級重點（4.6.1 → 4.8）

1. **每個 Session 的並發請求限制（.NET 4.7）**
   - 4.7 起，ASP.NET 對同一個 Session 的併發請求加入節流，預設上限為 50。
   - 若同 Session 有大量並發（例如多分頁或頻繁重新整理），可能出現 HTTP 500 並記錄警告。
   - 如需暫時還原舊行為，可於 `web.config` 加入：
```xml
<appSettings>
  <add key="aspnet:RequestQueueLimitPerSession" value="2147483647"/>
</appSettings>
```
   - 官方說明：
     [Throttle concurrent requests per session](https://learn.microsoft.com/en-us/dotnet/framework/migration-guide/retargeting/4.7.x#throttle-concurrent-requests-per-session)

2. **`HttpRuntime.AppDomainAppPath` 例外型別變更（.NET 4.6.2）**
   - 在 4.6.2，當 AppDomain 路徑含 null 字元時，可能拋出 `NullReferenceException`（舊版常見為 `ArgumentNullException`）。
   - 若程式有依例外型別分流處理，需要特別檢查。
   - 官方說明：
     [HttpRuntime.AppDomainAppPath Throws a NullReferenceException](https://learn.microsoft.com/en-us/dotnet/framework/migration-guide/retargeting/4.6.x#httpruntimeappdomainapppath-throws-a-nullreferenceexception)

3. **`HtmlTextWriter` 對 `<br/>` 輸出行為修正（.NET 4.6）**
   - 4.6 起，`RenderBeginTag`/`RenderEndTag` 對 `<br/>` 的輸出改為正確行為（只輸出一次）。
   - 若舊程式曾依賴過去的重複輸出行為，版面可能出現差異。
   - 官方說明：
     [HtmlTextWriter does not render br element correctly](https://learn.microsoft.com/en-us/dotnet/framework/migration-guide/retargeting/4.6.x#htmltextwriter-does-not-render-br-element-correctly)

4. **ASP.NET 可及性（Accessibility）改善（.NET 4.7.1）**
   - 4.7.1 起，ASP.NET Web Controls 與 Designer 的可及性互動有改善。
   - 一般不屬於破壞性變更，但若有 UI 自動化測試、輔助工具相依或客製控制項，建議加強回歸測試。
   - 官方說明：
     [ASP.NET Accessibility Improvements in .NET Framework 4.7.1](https://learn.microsoft.com/en-us/dotnet/framework/migration-guide/retargeting/4.7.x#aspnet-accessibility-improvements-in-net-framework-471)

---

## 4. 升級步驟

```
Step 1. 測試環境修改設定檔（參考第2節）[完成日期：2026/05/05]
Step 2. 測試環境執行功能測試（參考第3節）
Step 3. 開發環境修改設定檔
Step 4. 開發環境執行功能測試
Step 5. 建置主機(TFSBUILD)安裝.net framework 4.8 SDK [完成日期：2026/04/16]
Step 6. 正式環境進行金絲雀部署，更新時間查詢主機(NETAPX179)[完成日期：2026/05/26]
Step 7. 正式環境進行A/B 部署，更新一半主機
Step 8. 正式環境部署後監控 Application Event Log，確認無異常
```

---

## 5. 正式環境檢查清單

- [ ] `web.config` `compilation targetFramework` 改為 `4.8`
- [ ] `web.config` `httpRuntime targetFramework` 改為 `4.8`
- [ ] `app.config` `supportedRuntime sku` 改為 `v4.8`
- [ ] 對外 HTTPS 呼叫測試正常
- [ ] Application Event Log 無錯誤

---

## 6. 參考資料

- [.NET Framework 4.6.x Retargeting Changes（Microsoft 官方）](https://learn.microsoft.com/en-us/dotnet/framework/migration-guide/retargeting/4.6.x)
- [.NET Framework 4.7.x Retargeting Changes（Microsoft 官方）](https://learn.microsoft.com/en-us/dotnet/framework/migration-guide/retargeting/4.7.x)
- [.NET Framework 4.8.x Retargeting Changes（Microsoft 官方）](https://learn.microsoft.com/en-us/dotnet/framework/migration-guide/retargeting/4.8.x)
- [AppContext Class（AppContextSwitchOverrides 說明）](https://learn.microsoft.com/en-us/dotnet/api/system.appcontext)
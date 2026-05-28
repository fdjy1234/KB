# SAMService & SAMEntity 雙框架支援評估報告

**報告日期：** 2026-04-22  
**評估範圍：** SAMService、SAMEntity 是否可同時支援 .NET 10 與 .NET Framework 4.8  
**評估角色：** 資深系統架構師  
**評估版本：** 1.0  

---

## 一、執行摘要 (Executive Summary)

| 項目 | 結論 |
|------|------|
| 目前框架版本 | .NET Framework **4.6.1**（均需先升至 4.8）|
| SAMEntity 多目標可行性 | ⚠️ 中等 — 技術上可行，但需第三方框架支援 |
| SAMService 多目標可行性 | 🔴 低 — System.Web 深度耦合，不做重構難以達成 |
| 建議優先策略 | 先升至 .NET 4.8，再評估 IntelliSys 框架相容性 |
| 整體風險等級 | **高** |

---

## 二、現況盤點

### 2.1 專案基本資訊

| 項目 | SAMEntity | SAMService |
|------|-----------|------------|
| 專案格式 | 舊式 `.vbproj` (ToolsVersion=15.0) | 舊式 `.vbproj` (ToolsVersion=15.0) |
| 目標框架 | `v4.6.1` | `v4.6.1` |
| 語言 | Visual Basic .NET | Visual Basic .NET |
| 輸出類型 | Class Library (DLL) | Class Library (DLL) |
| 原始碼檔案數量 | ~50 個 XSD + Designer.vb | ~80 個 .vb 業務邏輯檔案 |
| OptionStrict | Off | On |
| 非同步模式 | 無 (Async/Await) | 無 (Async/Await) |

### 2.2 SAMEntity 相依套件

```
IntelliSys.NetExpress.Entity.dll  [v4.6.1 — 自製框架]
System                            [BCL]
System.Data                       [BCL]
System.Data.DataSetExtensions     [BCL]
System.Web.Services               [⚠️ 不支援 .NET 10]
System.Xml                        [BCL]
```

**Entity 類別結構：**  
所有 Entity 類別 (`beSAMxxxxxx`) 均為 `System.Data.DataSet` 的型別化子類別（Typed DataSet），由 `MSDataSetGenerator` 工具從 `.xsd` 自動產生。  
繼承鏈：`beSAMxxxxxx` → `System.Data.DataSet`

### 2.3 SAMService 相依套件

```
CommonLib.dll                           [v4.6.1 — 自製框架]
IntelliSys.NetExpress.DAO.dll           [v4.6.1 — 自製框架]
IntelliSys.NetExpress.Entity.dll        [v4.6.1 — 自製框架]
IntelliSys.NetExpress.Service.dll       [v4.6.1 — 自製框架]
IntelliSys.NetExpress.Service.Enterprise.dll [v4.6.1 — 自製框架]
IntelliSys.NetExpress.ServiceLib.dll    [v4.6.1 — 自製框架]
IntelliSys.NetExpress.Util.dll          [v4.6.1 — 自製框架]
IntelliSys.NetExpress.Web.dll           [v4.6.1 — 自製框架 含 Web!]
WorkFlowEntity.dll                      [v4.6.1 — 自製框架 (runtime路徑)]
System.Web                              [🔴 不存在於 .NET 10]
System.Web.Services                     [⚠️ 不支援 .NET 10]
System.Data                             [BCL]
System.Data.DataSetExtensions           [BCL]
System.Xml                              [BCL]
```

**Service 類別繼承鏈：**
```
大多數 Service 類別
  └── bsSAMBaseAppService
        └── CommonLib.Service.BaseAppService    ← 關鍵耦合點
        
部分 Email/Batch 類別
  └── IntelliSys.NetExpress.Service.BaseAppService (直接繼承)

bsFormula
  └── PUBService.BaseInvFormulaAgentService     ← 跨模組相依
```

---

## 三、多目標支援技術阻礙分析

### 3.1 🔴 阻礙等級：致命 (Critical Blockers)

#### CB-01：專案檔格式不支援多目標編譯

**影響範圍：** SAMEntity、SAMService（兩者均受影響）  
**問題說明：**  
目前使用**舊式 MSBuild `.vbproj` 格式**（`ToolsVersion="15.0"`），這種格式本身**不支援** `<TargetFrameworks>` 多目標屬性。必須先將專案格式遷移至 **SDK-style 格式**，才能使用以下語法：

```xml
<!-- 舊式格式（現況）— 不支援多目標 -->
<Project ToolsVersion="15.0" xmlns="...">
  <TargetFrameworkVersion>v4.6.1</TargetFrameworkVersion>

<!-- SDK-style 格式（所需）— 支援多目標 -->
<Project Sdk="Microsoft.NET.Sdk">
  <TargetFrameworks>net48;net10.0</TargetFrameworks>
```

**舊式 → SDK-style 遷移必要變更清單：**
- 移除所有 `<Import>` 標準 props/targets 行
- 移除所有 `<Compile Include="...">` 項目（SDK-style 自動掃描）
- 移除 `<Folder Include="My Project\">` 項目
- 轉換 `<Reference>` 為 `<PackageReference>` 或保留 HintPath 參考
- 對應 `AssemblyInfo.vb` → 移入 `<PropertyGroup>` 屬性

---

#### CB-02：System.Web 在 .NET 10 中不存在

**影響範圍：** SAMService（4 處程式碼直接調用）  
**問題說明：**  
`System.Web` 命名空間整個在 .NET Core/.NET 5+ 中被移除。以下為已確認的使用位置：

| 檔案 | 行號 | 使用的 API | 替代方案 |
|------|------|-----------|---------|
| `bsFormula.vb` | 375-376 | `System.Web.HttpCookie` + `HttpContext.Current.Response.Cookies.Add()` | `IHttpContextAccessor` + `IResponseCookies` |
| `bsFormula.vb` | 228 | `System.Web.HttpContext.Current.Session` (已被註解) | `ISession` |
| `bsFormula.vb` | 374 | `System.Web.HttpUtility.UrlEncode` (已被註解) | `Uri.EscapeDataString()` 或 `WebUtility.UrlEncode()` |
| `bsSAM302000Get.vb` | 298, 314, 316 | `System.Web.HttpRuntime.Cache.Get/Insert` | `IMemoryCache` (Microsoft.Extensions.Caching.Memory) |
| `bsSAM202000Adjust.vb` | 653-655 | `System.Web.HttpRuntime.Cache.Remove` | `IMemoryCache.Remove()` |

**架構衝擊：** `HttpRuntime.Cache` 是靜態全域存取，但 .NET 10 的 `IMemoryCache` 需透過 DI 注入。`BaseAppService` 若封裝了存取入口，則需要修改框架基底類別。

---

#### CB-03：IntelliSys.NetExpress 自製框架均為 .NET Framework 4.6.1

**影響範圍：** SAMEntity、SAMService（均深度依賴）  
**問題說明：**  
共有 **8 個自製 DLL**，全部版本標記為 `Version=4.6.1.0`，以 `HintPath` 方式從 `..\..\Lib\` 目錄載入。這些 DLL 在 .NET 10 下能否載入，取決於其內部是否使用了 .NET Framework 專屬 API（如 `System.Web`、`AppDomain`、`System.Runtime.Remoting` 等）。

**關鍵問題：**
- 這些 DLL 原始碼是否由 IntelliSys 提供？
- IntelliSys 是否有計畫提供 .NET Standard 2.0 或 .NET 6+ 版本？
- 這些 DLL 是否有 `netstandard2.0` 兼容的重新編譯版本？

若 IntelliSys 無法提供相容版本，則**唯一選擇**是替換整個框架（參見第五節替代方案）。

---

#### CB-04：System.Web.Services 的 Web Service 技術在 .NET 10 中廢除

**影響範圍：** SAMEntity（有 `System.Web.Services` 參考）  
**問題說明：**  
`System.Web.Services` 在 .NET Core 中被移除。若 Entity 類別有繼承或使用 ASMX Web Service 相關類別（`WebService`、`WebMethod` 屬性），需要遷移至：
- WCF（透過 [CoreWCF](https://github.com/CoreWCF/CoreWCF) 開源實作）
- ASP.NET Core 最小 API（REST）
- gRPC

---

### 3.2 ⚠️ 阻礙等級：重要 (Important Blockers)

#### IB-01：Typed DataSet 工具鏈在 .NET 10 SDK-style 中受限

**影響範圍：** SAMEntity（所有 .xsd/.xsc/.xss 檔案）  
**問題說明：**  
`MSDataSetGenerator` 是 VS 設計工具，在 SDK-style 專案中的整合度不如舊式專案。`.xsd` 檔案不再自動觸發 `Designer.vb` 重新生成。雖然 `System.Data.DataSet` 本身在 .NET 10 中仍然存在，但設計時工具的使用體驗會降低。

**緩解措施：** 可手動執行 `xsd.exe` 工具，或鎖定 Designer.vb 不再修改（視業務需求調整）。

---

#### IB-02：WorkFlowEntity.dll 路徑依賴執行期相對路徑

**影響範圍：** SAMService  
**問題說明：**  
`WorkFlowEntity.dll` 使用執行期路徑 `..\..\WebRoot\WebRootWeb\bin\`，而非標準 Lib 目錄。這在多目標建置環境中可能導致參考解析失敗。

---

#### IB-03：Oracle 特定 SQL 語法（非可攜性問題，但需注意）

**影響範圍：** SAMService  
**問題說明：**  
大量使用 Oracle 特有語法：
- `SEQ_SAMT_PLY_SHARE.NEXTVAL`（Oracle Sequence）
- `to_date('yyyyMMdd', 'yyyyMMdd')`（Oracle Date 函式）
- `/*+RULE*/`（Oracle Query Hint）

這些不影響框架升級，但若未來有更換資料庫計畫，需一併處理。

---

#### IB-04：VB.NET My 命名空間在 .NET 10 中部分受限

**影響範圍：** SAMService、SAMEntity  
**問題說明：**  
`My.Resources`、`My.Settings`、`My.Application` 等在 .NET 10 VB.NET 中有部分限制。SR.vb 使用 `System.Resources.ResourceManager` 直接讀取 `.resx`，相對安全，但需確認所有資源存取方式。

---

### 3.3 ✅ 相對安全項目

| 項目 | 說明 |
|------|------|
| `System.Data.DataSet`、`DataRow`、`DataTable` | .NET 10 完整支援 |
| `System.Data.IDbTransaction` | .NET 10 完整支援 |
| `System.Xml.XmlTextReader` | .NET 10 完整支援 |
| 物件序列化 `[Serializable]` | .NET 10 仍支援（BinaryFormatter 廢除，但 DataSet XML 序列化無影響）|
| `System.IO.StringReader` | .NET 10 完整支援 |
| `OptionStrict On/Off`、`OptionInfer` | .NET 10 VB.NET 支援 |
| 標準 SQL（SELECT/INSERT/UPDATE/DELETE）| 不受影響 |

---

## 四、可行性評估矩陣

### 4.1 SAMEntity 多目標可行性

| 阻礙項目 | 嚴重性 | 解決難度 | 解決方向 |
|---------|--------|---------|---------|
| CB-01 專案格式 | 🔴 致命 | 中 | 轉換 SDK-style |
| CB-03 IntelliSys.Entity.dll | 🔴 致命 | 高 | 依賴供應商 |
| CB-04 System.Web.Services | 🟠 重要 | 中 | 移除或使用 CoreWCF |
| IB-01 TypedDataSet 工具鏈 | 🟡 一般 | 低 | 接受限制 |

**綜合評分：中等可行（35%）**  
若 IntelliSys 提供相容 .dll，SAMEntity 因為業務邏輯純粹（純 DataSet 資料結構定義），遷移阻礙相對較小。

---

### 4.2 SAMService 多目標可行性

| 阻礙項目 | 嚴重性 | 解決難度 | 解決方向 |
|---------|--------|---------|---------|
| CB-01 專案格式 | 🔴 致命 | 中 | 轉換 SDK-style |
| CB-02 System.Web | 🔴 致命 | 高 | 重構架構 + DI |
| CB-03 IntelliSys 全套框架 | 🔴 致命 | 極高 | 依賴供應商或替換 |
| CB-04 System.Web.Services | 🟠 重要 | 中 | 移除或使用 CoreWCF |
| IB-02 WorkFlowEntity 路徑 | 🟡 一般 | 低 | 修改建置輸出 |

**綜合評分：低可行（10%，不重構則趨近 0%）**  
SAMService 對 `System.Web` 有直接業務邏輯依賴（非僅參考），且底層框架 (`BaseAppService`) 本身封裝了 Web 相關能力，若不修改 IntelliSys 框架或替換它，根本無法在 .NET 10 下編譯。

---

## 五、遷移策略建議

### 策略 A：真正多目標編譯（SDK-style multi-targeting）
**適合：** 有取得 IntelliSys 框架原始碼或供應商願意配合的情況

```
net48;net10.0
├── net48 目標：保留現有所有程式碼
└── net10.0 目標：
    ├── 使用 #If NETFRAMEWORK / #If NET 條件編譯
    ├── System.Web.HttpRuntime.Cache → IMemoryCache
    ├── System.Web.HttpCookie → IResponseCookies
    └── IntelliSys 框架 → 需要 netstandard2.0 相容版本
```

**優點：** 單一程式碼庫，一次建置兩個目標  
**缺點：** 需 IntelliSys 配合，條件編譯維護複雜度高  
**工期估算：** 6-12 個月（含框架評估）

---

### 策略 B：漸進式分層遷移（Adapter Pattern）
**適合：** IntelliSys 框架無法取得相容版本的情況（**推薦策略**）

```
現有 SAMEntity (net48)    ──────────────────────┐
現有 SAMService (net48)   ──────────────────────┤
                                                 ↓
                             SAM.Contracts (netstandard2.0)
                             [定義 Interface 合約]
                                                 ↓
                             SAM.Service.Core (net10.0)
                             [重新實作 Business Logic]
                             [使用 EF Core / Dapper / ADO.NET]
                             [不依賴 IntelliSys 框架]
```

**優點：** 不需 IntelliSys 配合，可逐功能遷移，風險分散  
**缺點：** 短期存在兩套程式碼，需設計清晰的合約介面  
**工期估算：** 按功能模組迭代，每模組 2-4 週

---

### 策略 C：先升級 .NET Framework 4.8（最低風險，推薦先行）
**適合：** 短期穩定、長期規劃遷移

```
現況：.NET Framework 4.6.1
   ↓（本次目標）
.NET Framework 4.8
   ↓（未來規劃）
評估 IntelliSys 相容性
   ↓
選擇策略 A 或 B
```

**升至 4.8 需要的變更（SAMEntity + SAMService）：**

```xml
<!-- SAMEntity.vbproj 與 SAMService.vbproj -->
<!-- 修改前 -->
<TargetFrameworkVersion>v4.6.1</TargetFrameworkVersion>

<!-- 修改後 -->
<TargetFrameworkVersion>v4.8</TargetFrameworkVersion>
```

同時需確認 `CommonLib.dll`、`IntelliSys.NetExpress.*.dll` 在 4.8 下無相容性問題（因版本標記為 4.6.1，但 .NET Framework 向上相容，應可正常運作）。

**優點：** 風險極低，工期短（1-2 天），可立即執行  
**缺點：** 未達多目標目標，但作為基礎步驟不可省略

---

## 六、具體修改清單

### 6.1 立即可執行（升至 .NET 4.8）

```
SAMEntity\SAMEntity.vbproj
  Line: <TargetFrameworkVersion>v4.6.1</TargetFrameworkVersion>
  改為: <TargetFrameworkVersion>v4.8</TargetFrameworkVersion>

SAMService\SAMService.vbproj  
  Line: <TargetFrameworkVersion>v4.6.1</TargetFrameworkVersion>
  改為: <TargetFrameworkVersion>v4.8</TargetFrameworkVersion>
```

---

### 6.2 為多目標準備（需 IntelliSys 配合後執行）

**Step 1：轉換 SAMEntity.vbproj 為 SDK-style**
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFrameworks>net48;net10.0</TargetFrameworks>
    <RootNamespace>SAMEntity</RootNamespace>
    <AssemblyName>SAMEntity</AssemblyName>
    <OptionStrict>Off</OptionStrict>
    <OptionInfer>On</OptionInfer>
    <OptionExplicit>On</OptionExplicit>
    <OptionCompare>Binary</OptionCompare>
  </PropertyGroup>

  <!-- 共同參考 -->
  <ItemGroup>
    <Reference Include="IntelliSys.NetExpress.Entity">
      <HintPath>..\..\Lib\IntelliSys.NetExpress.Entity.dll</HintPath>
    </Reference>
  </ItemGroup>

  <!-- 僅 .NET Framework 的參考 -->
  <ItemGroup Condition="'$(TargetFramework)' == 'net48'">
    <Reference Include="System.Web.Services" />
    <Reference Include="System.Data.DataSetExtensions" />
  </ItemGroup>

  <!-- .NET 10 的替代套件 -->
  <ItemGroup Condition="'$(TargetFramework)' == 'net10.0'">
    <PackageReference Include="System.Data.Common" Version="4.3.0" />
  </ItemGroup>
</Project>
```

**Step 2：SAMService 中 System.Web 使用點重構**

`bsSAM302000Get.vb` — Cache 抽象化
```vb
' 修改前（System.Web.HttpRuntime.Cache）
System.Web.HttpRuntime.Cache.Insert(cacName, dtdgdMain_Table, Nothing, ...)
dtdgdMain_Table = CType(System.Web.HttpRuntime.Cache.Get(cacName), ...)

' 修改後（注入 IMemoryCache 或使用靜態快取抽象）
#If NETFRAMEWORK Then
    System.Web.HttpRuntime.Cache.Insert(cacName, dtdgdMain_Table, Nothing, ...)
#Else
    _memoryCache.Set(cacName, dtdgdMain_Table, TimeSpan.FromMinutes(30))
#End If
```

`bsFormula.vb` — Cookie 操作抽象化
```vb
' 修改前
Dim cok As New System.Web.HttpCookie("sam202", sbJS.ToString)
System.Web.HttpContext.Current.Response.Cookies.Add(cok)

' 修改後（需要透過 IHttpContextAccessor 注入）
#If NETFRAMEWORK Then
    Dim cok As New System.Web.HttpCookie("sam202", sbJS.ToString)
    System.Web.HttpContext.Current.Response.Cookies.Add(cok)
#Else
    _httpContextAccessor.HttpContext.Response.Cookies.Append("sam202", sbJS.ToString)
#End If
```

---

### 6.3 需釐清事項（待確認後才能規劃）

| 確認項目 | 負責方 | 優先度 |
|---------|--------|--------|
| IntelliSys.NetExpress 是否有 .NET Standard 2.0 以上版本 | IntelliSys 供應商 | 🔴 最高 |
| IntelliSys.NetExpress.Web.dll 是否使用 System.Web 內部類別 | 架構師 + IntelliSys | 🔴 最高 |
| CommonLib.Service.BaseAppService 是否使用 System.Web | 架構師 | 🔴 最高 |
| WorkFlowEntity.dll 是否有 .NET 10 相容計畫 | 相關系統負責人 | 🟠 高 |
| SAMEntity 中 System.Web.Services 實際使用位置 | 架構師 | 🟠 高 |
| bsSAM301000Email.vb 電子郵件服務的 SMTP 送信邏輯 | 架構師 | 🟡 中 |

---

## 七、風險評估

| 風險 | 機率 | 衝擊 | 風險值 |
|------|------|------|--------|
| IntelliSys 無法提供相容框架版本，導致需全面替換 | 中 (40%) | 極高 | 🔴 高 |
| System.Web.HttpRuntime.Cache 被大量使用（程式碼以外範圍） | 中 (30%) | 高 | 🟠 中高 |
| Typed DataSet 的 Designer.vb 在 SDK-style 中無法再生 | 高 (70%) | 中 | 🟡 中 |
| VB.NET 在 .NET 10 的某些 Language Feature 不支援 | 低 (15%) | 中 | 🟡 低中 |
| Oracle ADO.NET 驅動在 .NET 10 的相容性 | 低 (10%) | 高 | 🟡 低中 |
| 編碼問題（Big5 vs UTF-8 程式碼檔案） | 低 (20%) | 低 | 🟢 低 |

---

## 八、建議行動計畫

### Phase 0：立即行動（本週內）
- [ ] **P0-1**：確認 IntelliSys 是否提供 .NET Standard 2.0+ 版本框架
- [ ] **P0-2**：將 SAMEntity 與 SAMService 目標框架從 `v4.6.1` 升至 `v4.8`
- [ ] **P0-3**：執行 `dotnet list package --vulnerable` 掃描套件漏洞（安全性）

### Phase 1：評估期（2-4 週）
- [ ] **P1-1**：反組譯 `CommonLib.dll` 確認 `BaseAppService` 是否依賴 `System.Web`
- [ ] **P1-2**：反組譯 `IntelliSys.NetExpress.Web.dll` 確認依賴程度
- [ ] **P1-3**：統計 SAMService 中全部 `System.Web` 使用點（已知 19 處，需完整列表）
- [ ] **P1-4**：評估 Typed DataSet 是否可逐步替換為 POCO + Dapper

### Phase 2：架構設計（1-2 個月，依 Phase 1 結果決定策略）
- [ ] **P2-1（策略A路線）**：設計 `#If NETFRAMEWORK` 條件編譯橋接方案
- [ ] **P2-2（策略B路線）**：設計 `SAM.Contracts (netstandard2.0)` 介面層
- [ ] **P2-3**：設計 `IMemoryCache` 替換 `HttpRuntime.Cache` 的抽象介面
- [ ] **P2-4**：確認 SDK-style 轉換對現有 CI/CD 建置流程的影響

### Phase 3：實施（3-6 個月，依模組優先排序）
- [ ] **P3-1**：轉換專案格式（SAMEntity 先行，風險較低）
- [ ] **P3-2**：逐模組重構 System.Web 使用點
- [ ] **P3-3**：整合測試（含 .NET 4.8 回歸測試）
- [ ] **P3-4**：整合測試（.NET 10 功能驗證）

---

## 九、結論

**SAMEntity** 具有中等可行性。其核心結構（Typed DataSet）在 .NET 10 中是技術可行的，主要阻礙是 IntelliSys.NetExpress.Entity.dll 的框架相容性問題，以及 System.Web.Services 的廢除問題，這兩項均有明確的替代方案。

**SAMService** 目前可行性低。除了框架相依外，程式碼中對 `System.Web.HttpRuntime.Cache` 和 `System.Web.HttpContext.Current` 的直接業務邏輯呼叫，代表著與 ASP.NET Web 管線的深度耦合。若要達到真正的多目標支援，**必須**重構這些呼叫點並引入 DI 抽象層，同時取得或替換整個 IntelliSys.NetExpress 框架。

**首要建議：** 在等待 IntelliSys 評估框架相容性的同時，立即執行升至 .NET 4.8 的低風險步驟，並進行深度程式碼審查（反組譯 `CommonLib.dll`），以取得後續策略決策所需的關鍵資訊。

---

*本報告基於 SAMService.vbproj、SAMEntity.vbproj 及所有 .vb 原始碼靜態分析所得。*  
*報告工具：GitHub Copilot (Claude Sonnet 4.6) — 資深系統架構師評估模式*

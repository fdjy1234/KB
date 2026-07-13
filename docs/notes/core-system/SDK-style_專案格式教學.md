# SDK-style 專案格式教學

**文件日期：** 2026-05-29  
**適用範圍：** .NET Framework 4.8、.NET 6+（含 .NET 10）  
**目標讀者：** 維護舊式 .csproj/.vbproj 的開發與架構人員

---

## 1. 什麼是 SDK-style

SDK-style 是 .NET Core 後導入的新專案檔格式，專案檔通常更短、更易維護，並支援現代化能力（例如多目標編譯、中央套件管理、跨平台 CLI 建置）。

最常見的宣告方式：

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
  </PropertyGroup>
</Project>
```

---

## 2. 舊式專案與 SDK-style 差異

| 項目 | 舊式專案（Non-SDK） | SDK-style |
|------|---------------------|-----------|
| 專案開頭 | `<Project ToolsVersion=... xmlns=...>` | `<Project Sdk="...">` |
| 原始碼檔案 | 需手動列 `<Compile Include=...>` | 預設自動包含 `**/*.cs` 或 `**/*.vb` |
| 套件管理 | `packages.config` 或舊式 `Reference` | `PackageReference` |
| 目標框架 | 多數為單一 `<TargetFrameworkVersion>` | 支援 `<TargetFrameworks>` 多目標 |
| Build 匯入 | 顯式 `<Import ...targets>` | 由 SDK 隱式處理 |
| 維護成本 | 高 | 低 |

---

## 3. 為什麼要轉 SDK-style

1. 專案檔更精簡，降低維護與衝突成本。
2. 讓同一份程式碼可同時建置 `net48` 與 `net10.0`。
3. 與 `dotnet` CLI、CI/CD、現代 NuGet 生態整合更好。
4. 便於導入 `Directory.Build.props`、`Directory.Packages.props` 等治理機制。

---

## 4. 最小可用範例

### 4.1 單目標（以 .NET Framework 4.8 為例）

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net48</TargetFramework>
    <LangVersion>latest</LangVersion>
    <Nullable>disable</Nullable>
  </PropertyGroup>
</Project>
```

### 4.2 多目標（同時支援 net48 + net10.0）

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFrameworks>net48;net10.0</TargetFrameworks>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>

  <ItemGroup Condition="'$(TargetFramework)' == 'net48'">
    <Reference Include="System.Web" />
  </ItemGroup>

  <ItemGroup Condition="'$(TargetFramework)' == 'net10.0'">
    <PackageReference Include="Microsoft.Extensions.Caching.Memory" Version="10.0.0" />
  </ItemGroup>
</Project>
```

---

## 5. 舊式專案轉換步驟（實務版）

### Step 1：先升級到可支援的最低目標

- 若目前仍是 .NET Framework 4.6.1，建議先升到 4.8 再做 SDK-style 轉換。
- 先確保現有功能測試可過，避免同時引入太多變因。

### Step 2：建立新 SDK-style 專案檔骨架

- 將開頭改為 `<Project Sdk="Microsoft.NET.Sdk">`。
- 將 `<TargetFrameworkVersion>v4.8</TargetFrameworkVersion>` 改為 `<TargetFramework>net48</TargetFramework>`。

### Step 3：移除舊式冗餘節點

- 移除大量 `<Compile Include="..." />`。
- 移除標準 `<Import Project="...Microsoft.CSharp.targets" />` 或 VB 對應匯入。
- 移除 `<Folder Include="..." />`（通常可省略）。

### Step 4：套件與參考整理

- `packages.config` 轉為 `PackageReference`。
- 保留必要的 DLL 參考（例如內部框架 DLL）並確認 `HintPath`。
- 若要雙框架，使用 `Condition` 區分不同目標的參考。

### Step 5：AssemblyInfo 策略

- 可保留現有 `Properties/AssemblyInfo.cs` 或 `My Project/AssemblyInfo.vb`。
- 若改由專案檔管理，加入：

```xml
<PropertyGroup>
  <GenerateAssemblyInfo>true</GenerateAssemblyInfo>
  <AssemblyTitle>MyProject</AssemblyTitle>
  <Version>1.0.0</Version>
</PropertyGroup>
```

### Step 6：建置與回歸測試

- 先本機建置，再跑關鍵回歸測試。
- 若是 Web 專案，確認 IIS / 發佈流程與既有部署管線相容。

---

## 6. 常見坑與排除建議

### 6.1 轉完後「檔案重複編譯」

原因：保留了舊式 `<Compile Include="...">`，又被 SDK 自動掃描到。  
處理：移除大多數手動 `<Compile>` 項目。

### 6.2 找不到 `System.Web`

原因：`System.Web` 只存在於 .NET Framework，不存在於 .NET 6+/10。  
處理：

1. 透過 `Condition` 僅在 `net48` 引用 `System.Web`。
2. 新框架改用 ASP.NET Core 對應能力（例如 `IMemoryCache`、`IHttpContextAccessor`）。

### 6.3 套件還在 `packages.config`

原因：尚未完成 NuGet 轉換。  
處理：改為 `PackageReference`，並確認套件版本是否支援目標框架。

### 6.4 既有第三方/內部 DLL 不支援 net10.0

原因：相依 DLL 僅提供 .NET Framework 版本。  
處理：

1. 優先向供應商取得 `netstandard2.0` 或 `net6+` 版本。
2. 在轉換期採「單目標 net48 + 分支重構」策略，避免一次到位造成風險過大。

---

## 7. 建議導入策略

1. **先求穩定：** 先把舊系統升到 `net48` + SDK-style（不急著上 net10.0）。
2. **再求並行：** 對低耦合模組導入多目標 `net48;net10.0`。
3. **最後替換：** 將 `System.Web` 及框架耦合點改寫為可注入抽象層。

---

## 8. 轉換檢查清單

- [ ] 專案檔已改為 `<Project Sdk="...">`
- [ ] `TargetFrameworkVersion` 已改為 `TargetFramework` 或 `TargetFrameworks`
- [ ] 大量 `<Compile Include>` 舊節點已移除
- [ ] `packages.config` 已轉 `PackageReference`
- [ ] 關鍵 DLL 參考路徑（HintPath）可解析
- [ ] `net48` 建置通過
- [ ] 若有多目標，`net10.0` 建置通過
- [ ] 關鍵回歸測試通過

---

## 9. 延伸閱讀

- [Migrate to SDK-style project format（Microsoft Learn）](https://learn.microsoft.com/nuget/resources/check-project-format)
- [Target frameworks in SDK-style projects（Microsoft Learn）](https://learn.microsoft.com/dotnet/standard/frameworks)
- [MSBuild reference for SDK-style projects（Microsoft Learn）](https://learn.microsoft.com/visualstudio/msbuild/msbuild-reference)

# SSRS / PBIRS 2025 條碼字型內嵌（Code39）處理結論

## 最後結論

在 Windows Server 2025 + SSRS/PBIRS 2025 環境中，手動安裝字型檔會放在「目前使用者目錄」。

要讓 Reporting Service 在匯出 PDF 時可讀取並內嵌條碼字型，必須同時滿足下列條件：

1. 字型檔實體存在於 `C:\Windows\Fonts`
2. 字型註冊存在於 `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Fonts`

本案例實測使用下列字型檔：

- `Code39_NS.ttf`
- `Code39ns_0.ttf`

完成上述兩步後，Reporting Service 才能讀到字型，PDF 才能成功內嵌條碼字型。

## 適用情境

- Visual Studio 預覽可正常顯示條碼
- 透過 SSRS/PBIRS 匯出 PDF 後，條碼變成純文字或 fallback 字型
- PDF 文件中看不到目標條碼字型，只看到 Microsoft Sans Serif、PMingLiU 等替代字型

## 問題成因說明

SSRS/PBIRS 的報表服務通常以服務帳號執行，而非目前登入的互動式使用者。
因此即使字型已安裝在目前使用者範圍，服務程序仍可能無法讀取，造成：

- 報表預覽正常，但伺服器端匯出異常
- PDF 產生時未內嵌目標字型
- 條碼欄位被替換為一般文字字型，導致掃描失敗

## 前提條件

在執行下列步驟前，建議先確認：

- 具備本機系統管理員權限
- 已取得正確的 Code39 字型檔來源
- 已知目前使用的服務名稱是 Power BI Report Server 或 SQL Server Reporting Services
- 報表欄位本身確實指定使用對應的條碼字型，而非設計端誤設為一般字型

## PowerShell 安裝腳本（系統層級）

> 請以系統管理員權限執行 PowerShell。

```powershell
# Code39 字型來源路徑
$fonts = @(
  @{ Source="C:\temp\font\Code39_NS.ttf";  FileName="Code39_NS.ttf";  RegName="Code39 NS (TrueType)" },
  @{ Source="C:\temp\font\Code39ns_0.ttf"; FileName="Code39ns_0.ttf"; RegName="Code39ns 0 (TrueType)" }
)

$fontsDir = "C:\Windows\Fonts"
$regPath = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Fonts"

foreach ($font in $fonts) {
  if (Test-Path $font.Source) {
    Copy-Item -Path $font.Source -Destination (Join-Path $fontsDir $font.FileName) -Force

    New-ItemProperty `
      -Path $regPath `
      -Name $font.RegName `
      -Value $font.FileName `
      -PropertyType String `
      -Force | Out-Null

    Write-Host "OK: $($font.FileName) -> $($font.RegName)"
  }
  else {
    Write-Host "MISSING: $($font.Source)"
  }
}

# 依實際服務名稱擇一重啟
Restart-Service PowerBIReportServer -Force
# Restart-Service SQLServerReportingServices -Force
```

## 驗證方式

完成安裝後，建議至少做以下驗證：

1. 確認字型檔已存在於 `C:\Windows\Fonts`
2. 確認登錄中存在對應項目：`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Fonts`
3. 重啟 PBIRS/SSRS 服務後重新匯出 PDF
4. 用 PDF 檢視器檢查文件屬性，確認已內嵌目標 Code39 字型
5. 實際使用條碼槍或解碼工具測試輸出內容是否可掃描

## 注意事項

- 若字型只存在 HKCU（目前使用者）而非 HKLM，SSRS/PBIRS 服務帳號通常無法使用
- 若只複製字型檔到 `C:\Windows\Fonts`，但未建立 HKLM 字型註冊，也可能仍然無法被服務辨識
- 若服務帳號、服務名稱或執行身分變更，建議重新驗證字型可見性與 PDF 內嵌結果
- 若報表使用的是字型顯示條碼，而非條碼控制項，還需確認資料是否包含 Code39 所需的起訖字元規則

## 常見誤區

- 只驗證設計端預覽成功，就誤判伺服器端匯出也會成功
- 以互動式使用者安裝字型，忽略 Windows Service 的執行脈絡
- 只看畫面顯示像條碼，未確認 PDF 實際是否內嵌正確字型
- 未重啟服務就直接重測，導致舊的字型快取或程序狀態仍在使用

## 補充建議

- 若是正式環境，建議將字型安裝步驟納入建置 SOP 或主機佈署文件
- 若環境有多台報表伺服器，需確認每台節點都完成相同的字型檔與登錄設定
- 若後續要升級或重建主機，建議一併保留字型來源檔與安裝腳本，避免靠人工回憶重建

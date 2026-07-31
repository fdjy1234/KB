# WSX壽險累保連線失敗處理：
## TLS 1.2 交握超時與「時好時壞」連線問題排查紀錄

---

## 1. 問題描述

在內部伺服器（WSAPX01）透過背景服務（以 `NT AUTHORITY\SYSTEM` 身分執行）呼叫壽險累保（10.210.253.10）時，發生 TLS 連線「時好時壞」的狀況（例如：連續發動 3 次，第 1 次成功，第 2、3 次失敗）。

外部伺服器使用的是**自發憑證 (Self-Signed Certificate)**。內部 .NET 程式碼中已加入略過憑證驗證的邏輯（`ServerCertificateValidationCallback = true`），且登錄檔已啟用 TLS 1.2，但連線依然極度不穩定。

## 2. 症狀與排查發現

### 2.1 系統事件檢視器 (Event Viewer)
在 WSAPX01 的「系統」日誌中，頻繁出現來源為 **Schannel** 的錯誤：
* **Event ID:** 36871
* **描述:** 在建立 TLS 用戶端認證時發生嚴重錯誤。
* **User:** `SYSTEM` (S-1-5-18)

### 2.2 Wireshark 封包分析
封包側錄檔案 WSAPX20260401.cap
透過擷取 WSAPX01 與 10.210.253.10 之間的封包，發現連線失敗時存在標準的**「超時斷線 (Timeout)」**特徵：
1.  用戶端送出 `Client Hello`。
2.  伺服器迅速回應 `Server Hello`（夾帶外部自發憑證）。
3.  **【異常點】** 用戶端隨後陷入長達 **10 秒鐘** 的完全靜默，未送出預期的 `Client Key Exchange`。
4.  10 秒後，外部伺服器因等待交握超時，主動發送 `TCP RST (Reset)` 強制切斷連線。

## 3. 根本原因分析 (Root Cause)

這個「薛丁格的連線 Bug」是由三個底層機制交織而成的結果：**Schannel 驗證機制**、**企業防火牆阻擋規則**，以及 **Windows 憑證負面快取**。

### 3.1 為什麼「程式寫了略過憑證」卻無效？
Windows 底層網路通訊組件 (Schannel) 的處理優先級高於 .NET 應用程式。當 Schannel 收到外部自發憑證時，因為不認識該憑證的發行者，作業系統預設會觸發**自動根憑證更新 (AuthRoot)**，試圖連上網際網路（微軟的更新伺服器）去查驗該憑證。如果連線在作業系統底層就崩潰，憑證根本不會交給 .NET 程式層，略過邏輯連執行的機會都沒有。

### 3.2 致命的 10 秒空窗期 (Firewall Drop)
在企業內網環境中，伺服器通常無法直接連外網。當 Schannel 發起對外驗證憑證的 HTTP 請求時，如果企業防火牆的規則是**默默丟棄 (Drop)** 封包而非主動拒絕 (Reject)，Schannel 就會卡在那裡死等回應。這個等待時間剛好約為 10 秒，最終導致外部伺服器等得不耐煩而強制斷線 (`TCP RST`)。

### 3.3 為什麼會「時好時壞」？(Windows 負面快取機制)
Windows 的加密 API (CAPI2) 具有**負面快取 (Negative Cache)** 機制，這解釋了為何有時連線會成功：
* **連線失敗（無快取 / 快取過期）：** Schannel 上網查驗 $\rightarrow$ 被防火牆 Drop $\rightarrow$ 傻等 10 秒 $\rightarrow$ 遭外部斷線 $\rightarrow$ 紀錄失敗狀態至快取。
* **連線成功（命中負面快取）：** Schannel 再次收到同張自發憑證 $\rightarrow$ 發現快取寫著「查不到」 $\rightarrow$ **放棄上網查驗**，瞬間將「憑證錯誤」狀態拋給上層 .NET $\rightarrow$ 觸發 .NET 程式中的略過憑證邏輯 $\rightarrow$ 強制放行，交握極速完成！

## 4. 解決方案

最根本且標準的解法是阻止 Windows 作業系統發起對外的憑證查驗請求。

**操作步驟：**
1.  取得外部單位的自發憑證檔案。
2.  在 WSAPX01 伺服器上，開啟本機電腦憑證管理員 (`certlm.msc`)。
3.  將該自發憑證匯入至 **「受信任的根憑證授權單位」 $\rightarrow$ 「憑證」**。

**結果：**
匯入後，Schannel 收到憑證會立刻在本機比對成功並信任，**直接放棄上網查詢**，從源頭避開了防火牆的 10 秒 Timeout。交握瞬間完成，問題徹底解決。

---

## 5. 架構與循序圖參考

### 5.1 系統架構圖
```mermaid
graph TD
    subgraph WSAPX01_伺服器_內部
        A[.NET 應用程式 - 略過憑證驗證邏輯]
        B[Windows Schannel / CAPI2 - 負責 TLS 交握]
        C[(Windows 內部負面快取)]
    end

    subgraph 外部網路環境
        D[外部目標伺服器 10.210.253.10]
        E[微軟 AuthRoot 或外部 CRL 發佈點]
    end

    F{企業核心防火牆}

    A -->|發起 HTTPS 請求| B
    B -.->|讀寫快取狀態| C

    B -->|1. 進行 TLS 交握| F
    F -->|正常放行| D
    
    B -.->|2. 嘗試上網查驗憑證| F
    F -.->|攔截並默默丟棄 Drop<br>導致 10 秒 Timeout| E
    
    classDef firewall fill:#e74c3c,stroke:#c0392b,color:white;
    class F firewall;
```

### 5.2 交握行為循序圖 (失敗與成功的情境對比)
```mermaid
sequenceDiagram
    participant App as WSAPX01 NET程式
    participant OS as Schannel 底層
    participant FW as 企業防火牆
    participant Ext as 外部伺服器
    participant WWW as 微軟驗證節點

    Note over App, WWW: 情境一：快取過期 (連線失敗) —— 導致 10 秒超時中斷
    App->>OS: 發起 HTTPS 請求
    OS->>Ext: TCP SYN & Client Hello
    Ext->>OS: Server Hello & 外部自發憑證
    OS->>OS: 檢查本機：不認識這張憑證
    OS->>FW: 發起 HTTP 請求 (準備驗證憑證)
    FW--x WWW: 封包被防火牆 Drop 掉 (無回應)
    Note over OS, FW: Schannel 傻等約 10 秒 (驗證流程卡死)
    Ext->>OS: TCP RST (等不到後續交握，強制斷線)
    OS->>OS: 記錄憑證驗證失敗至「負面快取」
    OS-->>App: 拋出底層通訊錯誤 (Event 36871)
    Note right of App: 交握在底層已中斷<br>.NET 的略過憑證邏輯無法執行
    
    Note over App, WWW: 情境二：命中負面快取 (連線成功) —— 時好時壞的原因
    App->>OS: 發起 HTTPS 請求
    OS->>Ext: TCP SYN & Client Hello
    Ext->>OS: Server Hello & 外部自發憑證
    OS->>OS: 檢查快取
    Note right of OS: 命中「負面快取」<br>知道對外查詢會 Timeout，直接放棄上網查！
    OS-->>App: 瞬間回報：憑證驗證失敗 (不受信任)
    Note right of App: 成功將控制權交還給程式碼
    App->>App: 觸發 Callback 邏輯 (return true)
    App->>OS: 強制放行 (略過錯誤)
    OS->>Ext: Client Key Exchange & Cipher Change Spec
    Note over OS, Ext: TLS 交握極速完成，開始傳輸 Application Data
```
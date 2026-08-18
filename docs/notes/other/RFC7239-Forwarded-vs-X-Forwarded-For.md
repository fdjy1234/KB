# RFC 7239 Forwarded 標頭 vs X-Forwarded-For

## 概述

`X-Forwarded-For` 是事實標準（de facto standard），而 `Forwarded` 是 RFC 7239 官方標準。兩者都用於在代理和負載均衡器環境中識別客戶端真實 IP，但在格式、功能和標準化程度上存在重要差異。

---

## RFC 7239 的 Forwarded 標頭

### 基本語法

```
Forwarded: [ parameter=value; ](, ...)
```

每個 `Forwarded` 標頭可包含以下參數（用分號分隔），多個代理跳轉用逗號分隔：

| 參數 | 說明 | 範例 |
|------|------|------|
| `for` | 客戶端 IP 地址 | `for=192.0.2.43` |
| `by` | 代理伺服器 IP 地址 | `by=203.0.113.60` |
| `host` | Host 請求標頭 | `host=example.com` |
| `proto` | 協議（http 或 https） | `proto=https` |

### 使用範例

#### 單一代理

```
Forwarded: for=192.0.2.43
```

#### 含協議和主機

```
Forwarded: for=192.0.2.43; proto=https; host=example.com
```

#### IPv6 地址（需括號和引號）

```
Forwarded: for="[2001:db8:cafe::17]"
```

#### 多個代理（各代理可有自己的參數）

```
Forwarded: for=192.0.2.43; proto=https, for=198.51.100.17; by=203.0.113.60; proto=http
```

---

## X-Forwarded-For 標頭

### 基本語法

```
X-Forwarded-For: client, proxy1, proxy2
```

標頭列出原始客戶端 IP，然後依次是每個轉發請求的代理。

### 使用範例

#### 單一 IP

```
X-Forwarded-For: 192.0.2.43
```

#### 多個代理

```
X-Forwarded-For: 192.0.2.43, 198.51.100.17, 203.0.113.43
```

#### IPv6 地址（無特殊格式要求）

```
X-Forwarded-For: 2001:db8:cafe::17
```

---

## 對比表

| 特性 | Forwarded (RFC 7239) | X-Forwarded-For |
|------|----------------------|-----------------|
| **標準化** | ✅ 官方 RFC 規範 | ❌ 事實標準 |
| **參數支援** | 🔷 豐富（for, by, host, proto） | 🔵 簡單（僅 IP） |
| **值格式** | `key=value; key=value` | `ip, ip, ip` |
| **IPv6 支援** | `for="[2001:db8::1]"`（需引號括號） | `2001:db8::1`（無特殊格式） |
| **可擴展性** | ✅ 易於添加新參數 | ❌ 難以擴展 |
| **協議資訊** | ✅ 包含 `proto` 參數 | ❌ 無法表達 |
| **代理 IP 記錄** | ✅ `by` 參數記錄 | ❌ 無法記錄 |
| **向後相容性** | ⚠️ 支援度尚未普及 | ✅ 廣泛支援 |

---

## 實際應用範例

### 場景：多層代理環境

```
客戶端 (192.0.2.43)
    ↓ HTTPS
代理 1 (198.51.100.17)
    ↓ HTTP
代理 2 (203.0.113.60)
    ↓ HTTP
Web 伺服器
```

**Web 伺服器收到的標頭對比：**

#### X-Forwarded-For（只有 IP）

```
X-Forwarded-For: 192.0.2.43, 198.51.100.17
```

**問題：**
- 不知道原始協議是 HTTPS
- 不知道每個代理的 IP
- 無法追蹤協議變化

#### Forwarded（完整資訊）

```
Forwarded: for=192.0.2.43; proto=https, 
          for=198.51.100.17; by=203.0.113.60; proto=http
```

**優勢：**
- 知道原始協議是 HTTPS
- 記錄了每個代理的 IP 地址
- 可以追蹤協議的變化過程

---

## 核心差異分析

### 何時使用各標頭

| 情況 | 推薦方案 | 原因 |
|------|---------|------|
| **僅需客戶端 IP** | X-Forwarded-For | 簡單易用 |
| **需要協議資訊** | Forwarded | 記錄 HTTPS/HTTP 變化 |
| **需要代理追蹤** | Forwarded | 記錄每個代理的 IP |
| **遺留系統相容性** | X-Forwarded-For | 廣泛支援 |
| **新專案/標準化** | Forwarded | 官方標準 |
| **過渡期** | 同時支援兩者 | 優先讀 Forwarded，備用 X-Forwarded-For |

### 訊息豐富度對比

**X-Forwarded-For 能告訴你：**
- ✅ 客戶端 IP
- ❌ 協議是什麼
- ❌ 經過哪些代理
- ❌ 主機名稱

**Forwarded 能告訴你：**
- ✅ 客戶端 IP
- ✅ 每跳的協議
- ✅ 每個代理的 IP
- ✅ 目標主機名稱
- ✅ 可擴展以容納新參數

---

## 實務建議

### 開發環境

| 階段 | 推薦策略 | 備註 |
|------|--------|------|
| **遺留系統維護** | 使用 X-Forwarded-For | 最大相容性 |
| **新專案開發** | 優先使用 Forwarded | RFC 7239 標準 |
| **過渡期** | 同時支援兩者 | 優先讀 Forwarded，有則使用，無則讀 X-Forwarded-For |
| **安全要求高** | 都只在受控代理環境信任 | 防止頭部欺騙 |

### 安全考量

⚠️ **重要：信任來源**

- **永遠不要盲目信任** `Forwarded` 或 `X-Forwarded-For` 標頭
- **僅在以下情況信任：**
  - 你控制該代理或負載均衡器
  - 代理正確配置為附加此標頭
  - 請求來自你信任的代理 IP
  - 防火牆規則限制不可信來源

### 實作檢查清單

```
□ 確認你的代理/負載均衡器支援的標頭類型
□ 在受控環境中配置代理轉發 IP
□ 實作 IP 源驗證（僅信任已知代理）
□ 記錄代理鏈的完整信息（便於調試）
□ 定期測試和驗證 IP 提取邏輯
□ 考慮同時支援 Forwarded 和 X-Forwarded-For
```

---

## 常見問題

### Q1: 我應該選擇哪一個？

**A:** 
- 如果是**新專案**，使用 `Forwarded`（RFC 7239）
- 如果需要**最大相容性**，使用 `X-Forwarded-For`
- 如果**兼容性要求高**，同時支援兩者

### Q2: 兩個標頭都存在時怎麼辦？

**A:** 優先使用 `Forwarded`，因為它是官方標準。如果 `Forwarded` 不存在或解析失敗，則退回到 `X-Forwarded-For`。

### Q3: IPv6 為什麼要用括號和引號？

**A:** 根據 RFC 7239，IPv6 地址包含冒號，必須用括號 `[]` 括起來以避免與參數分隔符混淆，再用引號包裝以確保符合 HTTP 標頭語法。

### Q4: 我可以信任這些標頭嗎？

**A:** 只有在以下條件都滿足時才能信任：
- 請求來自你已知且信任的代理
- 代理配置正確
- 防火牆阻止不可信源直接訪問你的應用

### Q5: 舊的代理軟體不支援 `Forwarded` 怎麼辦？

**A:** 同時實作兩個標頭的支援。大多數現代代理和負載均衡器都開始支援 RFC 7239。

---

## 參考資源

- [RFC 7239: Forwarded HTTP Extension](https://tools.ietf.org/html/rfc7239)
- [MDN: X-Forwarded-For](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For)
- [MDN: Forwarded](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Forwarded)

---

**最後更新：** 2026-08-18  
**文件用途：** HTTP 代理標頭選型和實作指南

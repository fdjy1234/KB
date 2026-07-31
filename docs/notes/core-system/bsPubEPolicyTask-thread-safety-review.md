# bsPubEPolicyTask Thread Safety 評估與改善建議

## 1. 文件目的

本文件針對 `bsPubEPolicyTask` 及其後續呼叫的電子保單服務進行 thread safety 靜態檢查，整理目前已確認的風險、可能影響，以及建議的改善方向，作為後續修正與程式碼評審依據。

## 2. 評估範圍

- 主程式: `PUB/PUBService/PUB/bsPubEPolicyTask.vb`
- 後續呼叫服務:
  - `CAR/CARService/TimeRay/bsCAREPolicy.vb`
  - `CAR/CARService/TimeRay/bsCAREPolicy_PN.vb`
  - `FIR/FIRService/B2B/bsFIREPolicy.vb`
  - `FIR/FIRService/B2B/bsFIREPolicy_TII.vb`
  - `CAS/CASService/EPolicy/bsCASEPolicy_R.vb`
  - `HAS/HASService/HasReport/bsHASEPolicy_R.vb`
  - `HAS/HASService/HasReport/bsHASEPolicy_R_PN.vb`

## 3. 結論摘要

目前程式存在明確的 thread safety 風險，主要集中在以下兩類:

1. `bsPubEPolicyTask` 的平行執行區塊中，存在閉包變數使用錯誤與共享狀態非同步更新問題。
2. 多個下游服務將每次請求的狀態存放在類別欄位，若服務執行個體被重用，會有跨執行緒資料污染風險。

其中最需要優先處理的是 `bsPubEPolicyTask` 的平行迴圈，因為這部分已經會直接影響保單號碼對應、成功失敗統計，以及錯誤判斷結果。

## 4. 發現清單

### 4.1 高風險: Task 閉包使用錯誤，可能讀到錯誤的 DataRow

- 位置:
  - `PUB/PUBService/PUB/bsPubEPolicyTask.vb:323`
  - `PUB/PUBService/PUB/bsPubEPolicyTask.vb:326`
- 現象:
  - 迴圈中先宣告 `Dim row = drTemp`，註解也明確指出要避免閉包抓錯資料。
  - 但 Task 內實際仍使用 `drTemp.Item(...)`，而不是 `row.Item(...)`。
- 風險說明:
  - 在 `For Each` 中建立多個 Task 時，閉包若持有外層變數 `drTemp`，就可能在 Task 實際執行時讀到不同列。
  - 結果可能出現保單號碼、險別、送件狀態對不上，甚至呼叫錯誤服務。
- 可能影響:
  - 送錯保單到錯誤的電子保單服務。
  - 錯誤訊息與實際保單不一致。
  - 成功失敗統計對錯筆資料。
- 建議改善:
  - 在進入 Task 前先建立不可變快照，例如:
    - `ipolicy`
    - `iinscls`
    - `isSend`
  - Task 內只使用這些區域變數，不再讀取 `drTemp` 或 `row`。

### 4.2 高風險: 多執行緒下對共享計數器做非原子遞增

- 位置:
  - `PUB/PUBService/PUB/bsPubEPolicyTask.vb:348`
  - `PUB/PUBService/PUB/bsPubEPolicyTask.vb:354`
  - `PUB/PUBService/PUB/bsPubEPolicyTask.vb:360`
- 現象:
  - `intSumOK += 1`
  - `inSumNotDo += 1`
  - `intSumNG += 1`
- 風險說明:
  - 這些整數在多個 Task 中被同時修改，`+= 1` 不是 thread-safe 操作。
  - 多個執行緒可能彼此覆蓋更新結果，造成計數遺失。
- 可能影響:
  - 統計結果小於真實值。
  - 異常件數被低估，導致通知或監控判斷失真。
- 建議改善:
  - 優先方案: 直接使用 `ConcurrentQueue` 的 `Count` 作為統計來源，避免雙重維護狀態。
  - 替代方案: 若仍需整數變數，改用 `Interlocked.Increment`。

### 4.3 中高風險: `isErr` 在平行區塊中被共享寫入

- 位置:
  - `PUB/PUBService/PUB/bsPubEPolicyTask.vb:359`
  - `PUB/PUBService/PUB/bsPubEPolicyTask.vb:440`
  - `PUB/PUBService/PUB/bsPubEPolicyTask.vb:441`
- 現象:
  - 在 Task `Catch` 中寫入 `isErr = True`。
  - 後面又有 `isErr = True '測試期，先固定給true使其發Mail`。
- 風險說明:
  - 多執行緒寫入共享布林值雖然最終多半會是 `True`，但設計上仍屬共享可變狀態。
  - 更大的問題是最終被固定設為 `True`，使錯誤旗標失去實際判斷意義。
- 可能影響:
  - 通知機制永遠發信，無法正確反映真實作業狀態。
  - 日後若有人依賴 `isErr` 判斷，容易誤判。
- 建議改善:
  - 直接以 `QueueSumNG.Count > 0` 或未結案狀態數量作為通知條件。
  - 移除平行區塊中的 `isErr` 寫入。
  - 若測試環境需要強制通知，應改用設定值或環境開關，不應硬編碼在正式邏輯中。

### 4.4 中高風險: `CheckJOB` 不是原子鎖，存在 TOCTOU 競態條件

- 位置:
  - `PUB/PUBService/PUB/bsPubEPolicyTask.vb:462`
- 現象:
  - 目前做法是查詢 `Jobs` 表，若符合條件的工作超過 1 筆則阻擋執行。
- 風險說明:
  - 這是典型的 Time Of Check To Time Of Use 問題。
  - 若兩個排程執行個體幾乎同時啟動，可能都在對方寫入穩定狀態前通過檢查。
- 可能影響:
  - 同一批資料被雙重處理。
  - 同保單重複送件。
  - 補償機制與日誌內容互相覆蓋。
- 建議改善:
  - 改成資料庫層級的原子鎖機制，例如:
    - 專用鎖表加唯一鍵
    - `SELECT ... FOR UPDATE NOWAIT`
    - 以單筆控制列做互斥鎖
  - 核心原則是「判斷是否能執行」與「取得執行權」必須在同一個原子動作內完成。

### 4.5 中高風險: 主服務本身使用類別欄位保存可變狀態

- 位置:
  - `PUB/PUBService/PUB/bsPubEPolicyTask.vb:32`
- 現象:
  - `Private htCmnCode As Hashtable`
  - 在 `DoRequest` 中被重新指派。
- 風險說明:
  - 若 `bsPubEPolicyTask` 的 service instance 會被框架重用，則不同請求可能共用同一個 `htCmnCode` 欄位。
  - 雖然目前此欄位只在主流程初始化時使用一次，但設計上仍屬於可變共享狀態。
- 建議改善:
  - 將 `htCmnCode` 改為 `DoRequest` 內的區域變數。
  - 原則上，service 類別應避免保存每次請求才決定的狀態。

### 4.6 高風險: 下游服務使用類別欄位保存請求狀態

#### 4.6.1 FIR 服務

- 位置:
  - `FIR/FIRService/B2B/bsFIREPolicy.vb:13`
  - `FIR/FIRService/B2B/bsFIREPolicy.vb:14`
  - `FIR/FIRService/B2B/bsFIREPolicy.vb:15`
  - `FIR/FIRService/B2B/bsFIREPolicy_TII.vb:15`
  - `FIR/FIRService/B2B/bsFIREPolicy_TII.vb:16`
  - `FIR/FIRService/B2B/bsFIREPolicy_TII.vb:17`
- 現象:
  - 使用類別欄位保存:
    - `trans`
    - `htCmnCode`
    - `sUserName`
- 風險說明:
  - 若同一個 service instance 被多個請求共用，這些欄位會互相覆寫。
- 建議改善:
  - 將上述欄位全部改為 `DoRequest` 內區域變數。

#### 4.6.2 CAS 服務

- 位置:
  - `CAS/CASService/EPolicy/bsCASEPolicy_R.vb:27`
  - `CAS/CASService/EPolicy/bsCASEPolicy_R.vb:28`
  - `CAS/CASService/EPolicy/bsCASEPolicy_R.vb:29`
  - `CAS/CASService/EPolicy/bsCASEPolicy_R.vb:30`
  - `CAS/CASService/EPolicy/bsCASEPolicy_R.vb:31`
- 現象:
  - 使用類別欄位保存:
    - `trans`
    - `htCmnCode`
    - `sUserName`
    - `bolTWCA`
    - `idbCon`
- 風險說明:
  - 這些都是請求過程中可變的狀態，若多請求共用 instance，風險更高。
  - 其中 `bolTWCA` 屬於流程分支控制旗標，被覆寫時會直接影響送件路徑。
- 建議改善:
  - 全部移為區域變數。
  - 若必須保存跨方法狀態，應改用方法參數傳遞，不應存在 service instance 欄位。

#### 4.6.3 HAS 服務

- 位置:
  - `HAS/HASService/HasReport/bsHASEPolicy_R.vb:20`
  - `HAS/HASService/HasReport/bsHASEPolicy_R.vb:21`
  - `HAS/HASService/HasReport/bsHASEPolicy_R.vb:22`
  - `HAS/HASService/HasReport/bsHASEPolicy_R_PN.vb:20`
  - `HAS/HASService/HasReport/bsHASEPolicy_R_PN.vb:21`
  - `HAS/HASService/HasReport/bsHASEPolicy_R_PN.vb:22`
- 現象:
  - 使用類別欄位保存:
    - `trans`
    - `htCmnCode`
    - `sUserName`
- 建議改善:
  - 與 FIR 相同，改為區域變數。

#### 4.6.4 CAR 服務

- 位置:
  - `CAR/CARService/TimeRay/bsCAREPolicy.vb:15`
  - `CAR/CARService/TimeRay/bsCAREPolicy.vb:16`
  - `CAR/CARService/TimeRay/bsCAREPolicy.vb:17`
  - `CAR/CARService/TimeRay/bsCAREPolicy.vb:18`
  - `CAR/CARService/TimeRay/bsCAREPolicy.vb:19`
  - `CAR/CARService/TimeRay/bsCAREPolicy_PN.vb:15`
  - `CAR/CARService/TimeRay/bsCAREPolicy_PN.vb:16`
  - `CAR/CARService/TimeRay/bsCAREPolicy_PN.vb:17`
  - `CAR/CARService/TimeRay/bsCAREPolicy_PN.vb:18`
  - `CAR/CARService/TimeRay/bsCAREPolicy_PN.vb:19`
  - `CAR/CARService/TimeRay/bsCAREPolicy_PN.vb:20`
  - `CAR/CARService/TimeRay/bsCAREPolicy_PN.vb:21`
- 現象:
  - 使用類別欄位保存:
    - `cf`
    - `RCVKIND`
    - `original`
    - `PrintFont`
    - `DPLYISSU`
    - `IPLYSEQ`
    - `IEDRSEQ`
- 風險說明:
  - 這類欄位在列印、資料轉換與流程判斷中常會被讀寫，若共用 instance，污染風險很高。
- 建議改善:
  - 全部改為區域變數。
  - 若有共用邏輯，應封裝在無狀態 helper method，而不是保存在欄位中。

## 5. 優先改善順序

### 第一優先: 立即修正 `bsPubEPolicyTask`

建議先處理以下項目，因為回歸範圍小、風險降低效果最大:

1. 修正 Task 閉包讀值方式。
2. 移除平行區塊中的非 thread-safe 計數器。
3. 讓通知條件改由 `QueueSumNG.Count` 與未結案數量決定。
4. 將 `htCmnCode` 改成區域變數。

### 第二優先: 下游服務無狀態化

依服務逐支調整，原則如下:

1. 類別欄位只保留真正不變的常數或唯讀設定。
2. 每次請求才會改變的資料，全部改成區域變數。
3. 若跨方法需要傳遞資料，透過參數或結構物傳遞。

### 第三優先: 排程互斥控制改為資料庫原子鎖

這一項屬於架構層改善，建議與 DBA 或排程框架負責人共同確認後導入。

## 6. 建議修正範例方向

### 6.1 平行區塊資料快照

建議模式如下:

```vb
For Each drTemp As DataRow In dsTemp.Tables(0).Rows
    Dim ipolicy As String = GetString(drTemp.Item("IPOLICY"))
    Dim iinscls As String = GetString(drTemp.Item("IINSCLS"))
    Dim isSend As String = GetString(drTemp.Item("IS_SEND"))

    tasks.Add(Task.Run(Async Function()
        Await semaphore.WaitAsync()
        Try
            ' Task 內只使用 ipolicy / iinscls / isSend
        Finally
            semaphore.Release()
        End Try
    End Function))
Next
```

### 6.2 計數方式統一

建議不要同時維護 Queue 與整數計數器，可改成:

```vb
Dim successCount As Integer = QueueSumOK.Count
Dim skipCount As Integer = QueueSumNotDo.Count
Dim errorCount As Integer = QueueSumNG.Count
```

若仍要整數遞增，則應使用:

```vb
Interlocked.Increment(intSumOK)
```

## 7. 補充說明

### 7.1 本次為靜態審查

本報告依據程式碼靜態閱讀與執行緒安全原則判斷，未實際連接資料庫或執行壓力測試。

### 7.2 下游服務風險成立的前提

下游服務的類別欄位風險，與服務生命週期有關:

- 若框架每次呼叫都建立全新 instance，風險會降低。
- 若框架會重用 instance，則目前設計具高風險。

即使目前剛好是 per-request instance，仍建議改為無狀態設計，避免未來框架調整、快取、物件池化或重構後出現隱性併發問題。

## 8. 建議執行方案

### 方案 A: 最小風險修補

先只修 `bsPubEPolicyTask`，內容包含:

1. 修正閉包變數。
2. 調整計數方式。
3. 改善通知條件。
4. 將 `htCmnCode` 區域化。

適合先快速降低實際風險。

### 方案 B: 完整無狀態化重構

除 `bsPubEPolicyTask` 外，同步調整 CAR/FIR/CAS/HAS 電子保單服務，將可變欄位全部改為區域變數。

適合一次性解決整個電子保單流程的 thread safety 問題。

## 9. 最終建議

若以系統穩定性與修正成本評估，建議採用以下順序:

1. 先修 `bsPubEPolicyTask` 的平行處理缺陷。
2. 再修下游服務的可變類別欄位。
3. 最後補上資料庫層級的排程互斥鎖。

以上三步完成後，整體電子保單排程的 thread safety 風險會明顯下降，且後續維運時較不容易出現難以重現的間歇性錯誤。
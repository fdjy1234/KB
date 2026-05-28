# 說明文件：curl -vk --resolve midev.hoan.com.tw:443:35.206.252.67 https://midev.hoan.com.tw/

簡短說明  
這個指令用 curl 向主機名 `midev.hoan.com.tw` 的 HTTPS 服務發出請求，但強制把該主機名解析為 IP `35.206.252.67`（不使用系統 DNS）。使用 `-v` 顯示詳細的 TLS / HTTP 訊息，`-k` 表示忽略伺服器憑證驗證錯誤（不建議在生產環境長期使用，只作診斷）。

---

## 完整指令
```bash
curl -vk --resolve midev.hoan.com.tw:443:35.206.252.67 https://midev.hoan.com.tw/
```

---

## 各參數逐項說明

- curl  
  命令本體，用於發 HTTP/HTTPS 請求。

- -v, --verbose  
  顯示詳細的傳輸/連線資訊（TCP、TLS 握手、HTTP header 等），方便偵錯。輸出會包含 schannel/openssl 的交握訊息、ALPN negotiation、TLS record 等。

- -k, --insecure  
  忽略伺服器憑證的驗證錯誤（例如自簽或不被信任的 CA）。curl 會仍然建立 TLS 連線，但不檢查憑證鏈或主機名稱是否被信任。

- --resolve <host:port:address>  
  告訴 curl：當要連到 `<host>` 的 `<port>` 時，直接使用 `<address>`（覆寫 DNS）。  
  特性與注意：
  - 只影響 curl 的此個請求（不改系統 DNS）。
  - curl 在 TLS 握手中仍會以 URL 中的主機名稱（`midev.hoan.com.tw`）做 SNI（Server Name Indication）與 HTTP `Host:` header，因此可以同時測試 SNI 行為但連到指定 IP。
  - 格式可以多次指定多個 host:port:address。

- 最後的 URL `https://midev.hoan.com.tw/`  
  指定要訪問的主機名（curl 會把它當作 Host header 與 SNI）。即便我們用 `--resolve` 指定 IP，URL 的主機名稱仍會被送出作為 SNI/Host。

---

## 為何使用這個指令（典型用途）
- 在不改系統 DNS 的情況下，測試針對特定 IP 與特定主機名（SNI）的 TLS 行為。  
- 驗證 SNI 是否正確送出（尤其在服務使用 SNI 分流時很重要）。  
- 偵測路徑中間設備（proxy / WAF / SSL inspection）是否攔截或修改流量。  
- 在內部測試或排除 DNS 問題時，快速把主機名指向測試 IP。

---

## 常見 verbose 輸出行與代表含義（範例與解釋）

範例輸出行：
- Added midev.hoan.com.tw:443:35.206.252.67 to DNS cache  
  → curl 已把這個解析條目暫存（來自 --resolve）。

- Trying 35.206.252.67:443...  
  → 正在對指定 IP 與 port 建立 TCP 連線。

- schannel: disabled automatic use of client certificate  
  → Windows 下 schannel 的訊息（表示不會自動用 client cert）。

- ALPN: curl offers http/1.1  
  → curl 告知 server 支援 http/1.1（ALPN extension）。

- ALPN: server accepted http/1.1  
  → server 在 ALPN 中選擇 http/1.1。

- Connected to midev.hoan.com.tw (35.206.252.67) port 443  
  → TCP 連線成功建立（但不代表 TLS 成功）。

- Recv failure: Connection was reset  
  → 收到連線重設（RST）或在 TLS 握手中被中斷。表示 TLS 握手無法完成（很可能是中間設備或 server 發出 RST）。

- schannel: failed to receive handshake, SSL/TLS connection failed  
  → TLS 握手失敗（在 Windows 下的 schannel stack 錯誤資訊）。

常見 curl 退出代碼（診斷用）
- 35：SSL connect error（例如 TLS 握手失敗或被重置）。  
- 7：Failed to connect to host（TCP 連線錯誤）。  
- 28：Operation timeout 等。

---

## 如何理解與對應你觀察到的錯誤（例如你現場遇到）
你曾看到：
- Failed to set TCP_KEEPINTVL ... errno 10042  
  → 嘗試設定某些 socket option（例如 KEEPALIVE 參數）在該平臺不被允許或不支援，通常只是警告，不致於造成握手失敗。

- Recv failure: Connection was reset / schannel: failed to receive handshake  
  → 表示在 TLS ClientHello 後收到了 TCP RST 或其它中斷，導致握手中止。常見原因：
  - Inline 中介（例如 Forcepoint / Palo Alto）基於 policy/SSL inspection 對該 ClientHello 重置連線。  
  - 中間設備與上游伺服器握手失敗，因而回復 RST 給 client。  
  - Server 端直接重設（較少見，通常會有後端日誌）。

- Windows Event: Schannel Event ID 36871 (internal error 10013)  
  → Schannel 回報握手相關的嚴重錯誤。若同時看到 network 層 RST，通常是握手被中途中斷導致該 event。

---

## 進階調試步驟（建議順序）

1. 用 curl 確認情況（你已做）：
   ```bash
   curl -vk --resolve midev.hoan.com.tw:443:35.206.252.67 https://midev.hoan.com.tw/
   ```

2. 若 TLS 握手被 reset，抓包（client 端）以確認時序、SYN/SYN-ACK/ACK、ClientHello、RST：
   - Windows（若使用 NetMon）或可用 Wireshark/tshark：
     - tshark -i <iface> -f "host 35.206.252.67 and port 443" -w capture.pcap
     - 或在已抓好的 pcap 上過濾：tshark -r capture.pcap -Y "tcp.flags.reset==1 or tls.handshake.type==1" -V

3. 檢查 ClientHello（是否有 SNI、哪些 cipher、是否有 ALPN 等）：
   - 在抓包中找到 ClientHello 的 hex 或用 tshark 解碼：
     - tshark -r capture.pcap -Y "tls.handshake.type==1" -T fields -e tls.handshake.extensions_server_name -e tls.handshake.ciphersuites -e tls.record.version

4. 若懷疑被中介攔截（inline proxy/SSL inspection）
   - 檢查封包中的以太層 MAC：若 RST 的 src MAC 為某中介設備的 MAC（而非後端 server 的 MAC），表示中介代發 reset。  
   - 把 pcap 與相關時間、source IP、destination IP、ClientHello hex 提交給 NetSec / 防火牆團隊，請他們在 appliance 上查 traffic/decryption logs，或將該 session 暫時白名單 bypass 以測試。

5. 用 openssl s_client 作替代測試（比較不同 TLS client 指紋）：
   ```bash
   openssl s_client -connect 35.206.252.67:443 -servername midev.hoan.com.tw -tls1_2 -msg
   ```
   - 如果 openssl 能通而 schannel/curl 不能，說明中介可能基於 ClientHello 指紋做判斷（不同 TLS library 會產生不同 ClientHello 指紋）。

6. 若要強制 curl 使用特定 TLS 版本或 cipher：
   - 強制 TLS1.2：
     ```bash
     curl --tlsv1.2 -vk --resolve midev.hoan.com.tw:443:35.206.252.67 https://midev.hoan.com.tw/
     ```
   - 指定 cipher（視 curl 與底層 TLS library 支援）：
     ```bash
     curl --ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384' -vk ...
     ```

7. 檢查 Windows 系統層：
   - Event Viewer → System → 搜尋 Schannel 相關 event，取得完整訊息（Event ID 與 internal state）。貼回來可供分析。
   - 檢查是否有系統代理 / 全站代理、或 endpoint security（AV）在做 HTTPS 檢查：
     - netsh winhttp show proxy
     - 檢查 IE/Edge 的網路代理設定（inetcpl.cpl → Connections → LAN settings）。

8. 若可，讓 NetSec 臨時 bypass SSL inspection（白名單 172.20.105.174 或 host），再重試 curl 以確定是否為中介所致。

---

## 與 SNI / Host header 有關的注意事項
- 使用 `--resolve` 並不會改變 URL 中的 host 字串；curl 仍會把 URL 的主機名放到 TLS SNI 與 HTTP Host header，因此這是檢測 SNI 行為的好方法。  
- 如果你把 URL 改成直接使用 IP（例如 `https://35.206.252.67/`），curl 在建立 TLS 時的 SNI 與 Host header 將是 IP，這通常會導致伺服器無法選到正確的 virtual host（若 server 以 SNI 做 vhost）。

---

## 安全與風險提醒
- `-k`（--insecure）會跳過憑證驗證，請僅在測試/診斷時使用，勿在生產或對敏感資料的請求中長期啟用。  
- 如果發現中介在做 TLS 介入（SSL inspection），請與 NetSec 討論是否要在該 client 安裝中介的根憑證（如果政策允許）或調整白名單策略。

---

## 範例：給 NetSec / 防火牆團隊的簡短檢查項目（可直接複製貼上）
- 時間（精確）：<填入測試時間>  
- Source IP/MAC：172.20.105.174 / <client MAC>  
- Destination IP/Host：35.206.252.67 (midev.hoan.com.tw) :443  
- 測試命令：`curl -vk --resolve midev.hoan.com.tw:443:35.206.252.67 https://midev.hoan.com.tw/`  
- 觀察：Client 完成 TCP 三次握手並送出 TLS ClientHello，隨即 client 收到 RST+ACK；client-side pcap 與 verbose 輸出已保存（附上）。請於 appliance（Palo Alto / Forcepoint）上：
  - 搜尋該 time window 的 Traffic / Decryption logs（filter by src/dst/port）。  
  - 匯出 appliance-side pcap 或 session log，並回報是否有 rule/policy 導致 reset（policy id / reason）。  
  - 建議短暫 bypass SSL inspection for this source/host，協助定位。

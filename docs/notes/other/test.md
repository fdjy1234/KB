1. 本機 PC 將 TFS 上 config 最新設定抓下來
2. config 進行更新修改，簽入 TFS
3. 透過【publish_configs.bat】發布各專案 config 設定檔
4. 透過【deploy_to_tnetap3.bat】將各專案 config 設定檔放置 tnetap3 各專案路徑中
5. 程式科通知系統科執行【更新批次檔】
6. 【更新批次檔】將 tnetap3 資料搬至對應主機 C 槽

```mermaid
flowchart LR
	subgraph TFS["🏢 TFS 版本庫"]
		T["TFS Config"]
	end

	subgraph Dev["👨‍💼 Config 管理員"]
		A["1️⃣ 從 TFS 下載最新 config"]
		B["2️⃣ 修改 config 並簽入 TFS"]
	end

	subgraph Batch1["📋 批次檔 1 publish_configs.bat"]
		C["3️⃣ 發布各專案 config 設定檔"]
	end

	subgraph Middle["🖥️ tnetap3 中繼主機"]
		D["4️⃣ 放置 config 於各專案路徑"]
	end

	subgraph Batch2["📋 批次檔 2 deploy_to_tnetap3.bat"]
		E["執行部署到 tnetap3"]
	end

	subgraph System["👨‍💻 系統科"]
		S["5️⃣ 收到通知並執行更新"]
	end

	subgraph Batch3["📋 批次檔 3 update_servers.bat"]
		F["執行更新批次"]
	end

	subgraph Target["🖥️ 主機 C 槽目標主機"]
		G["6️⃣ 將 tnetap3 資料搬至對應主機 C 槽"]
	end

	T --> A --> B --> C
	C --> D --> S
	D --> E --> F --> G

	style Dev fill:#e1f5ff
	style System fill:#c8e6c9
	style Middle fill:#fff9c4
	style Target fill:#ffccbc
```

```mermaid
sequenceDiagram
	autonumber
    participant TFS as TFS<br/>(版本庫)
	participant DevPC as 本機PC<br/>(Config管理員)
	participant Pub as 編譯各主機組態檔<br/>(publish_configs.bat)
	participant Dep as 發佈組態檔至tnetap3<br/>(deploy_to_tnetap3.bat)
    participant TNetAP3 as tnetap3<br/>(中繼主機)
	participant Sys as 程式更新人員<br/>(系統科)
	participant Upd as 程式更新批次檔<br/>(update_servers.bat)
	participant Target as 目標主機<br/>(C槽)

	DevPC->>TFS: 1. 抓取最新 config
	DevPC->>TFS: 2. 修改後簽入 config
	DevPC->>Pub: 3. 執行 publish_configs.bat
	Pub->>TNetAP3: 發布各主機 config 設定檔
	DevPC->>Dep: 4. 執行 deploy_to_tnetap3.bat
	Dep->>TNetAP3: 放置 config 到各專案路徑
	DevPC->>Sys: 5. 通知執行更新批次檔
	Sys->>Upd: 執行 update_servers.bat
	Upd->>Target: 6. 將 tnetap3 資料搬至各主機C槽
```

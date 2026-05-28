「InvokeBusinessService 呼叫後的循序圖（sequenceDiagram）」與「整體架構圖（architecture diagram）」。圖中以程式流程與主要物件為主，突顯呼叫路徑與資料回傳。
簡短說明：
•	循序圖：顯示從 InvokeBusinessService(string, DataSet) 到最終 Request(string, DataSet) 的同步呼叫流程與回傳。
•	架構圖：展示各個元件與其關係（CallContext 傳遞、BSAgent 封裝、BSServer 建立與 ServiceUtil 產生 service 實例）。

```mermaid
graph TD
  Client["Web Client"]
  NEWebService["IntelliSys.NetExpress.Web.NEWebService"]
  BSAgent["IntelliSys.NetExpress.Web.BSAgent"]
  CallContext["System.Runtime.Remoting.Messaging.CallContext(CallContext.SetData)"]
  BSServer["IntelliSys.NetExpress.Server.BSServer"]
  ServiceUtil["IntelliSys.NetExpress.ServiceLib.ServiceUtil"]
  Service["IBusinessService(concrete implementation)"]

  Client --> NEWebService
  NEWebService --> BSAgent
  BSAgent --> CallContext
  BSAgent --> BSServer
  BSServer --> ServiceUtil
  ServiceUtil --> Service
  BSServer --> Service
  Service --> BSServer
  BSServer --> BSAgent
  BSAgent --> NEWebService
  NEWebService --> Client
```

```mermaid
sequenceDiagram
  participant Client as "Web Client"
  participant NEWebService as "NEWebService"
  participant BSAgent as "BSAgent"
  participant CallContext as "CallContext"
  participant BSServer as "BSServer"
  participant ServiceUtil as "ServiceUtil"
  participant Service as "IBusinessService (concrete)"

  Client->>NEWebService: InvokeBusinessService(...)
  NEWebService->>NEWebService: createCallContextData()
  NEWebService->>BSAgent: InvokeBusinessService(..., ccd)
  BSAgent->>CallContext: CallContext.SetData(...)
  BSAgent->>BSServer: new BSServer() / Request(serviceName...)
  BSServer->>BSServer: SetServiceCulture() / setHostTrace()
  BSServer->>ServiceUtil: CreateServiceObject(serviceName)
  ServiceUtil-->>Service: 返回 service 實例
  BSServer->>Service: svc.Request(ds)
  Service-->>BSServer: DataSet result
  BSServer-->>BSAgent: return rt
  BSAgent-->>NEWebService: return rt
  NEWebService-->>Client: return DataSet
```

```mermaid
sequenceDiagram
    participant Client as "Web Client"
    participant NEWebService as "NEWebService"
    participant BSAgent as "BSAgent"
    participant CallContext as "CallContext"
    participant Remoting as "Remoting"
    participant BSServer as "BSServer"
    participant ServiceUtil as "ServiceUtil"
    participant Service as "IBusinessService"

    Client->>NEWebService: InvokeBusinessService
    NEWebService->>NEWebService: createCallContextData
    NEWebService->>BSAgent: InvokeBusinessService
    BSAgent->>CallContext: CallContext.SetData
    BSAgent->>Remoting: EnsureRemotingConfigured
    alt remote BSServer available
        BSAgent->>Remoting: Activator.GetObject -> proxy
        BSAgent->>BSServer: proxy.Request
    else fallback to local
        BSAgent->>BSServer: new BSServer() + Request
    end
    BSServer->>BSServer: SetServiceCulture / setHostTrace
    BSServer->>ServiceUtil: CreateServiceObject
    ServiceUtil-->>Service: return instance
    BSServer->>Service: svc.Request
    Service-->>BSServer: DataSet result
    BSServer-->>BSAgent: return DataSet
    BSAgent-->>NEWebService: return DataSet
    NEWebService-->>Client: return DataSet
```

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> BuildCallContext : request arrives
    BuildCallContext --> BSAgentInvoking

    state BSAgentInvoking {
        [*] --> CheckRemotingConfig
        CheckRemotingConfig --> RegisterChannels : if remoting enabled
        CheckRemotingConfig --> SetCallContext : if not enabled
        RegisterChannels --> SetCallContext
        SetCallContext --> CreateProxyOrLocal
        CreateProxyOrLocal --> CallBSServer
        CallBSServer --> [*]
    }

    state BSServerSide {
        [*] --> Init
        Init --> CreateServiceInstance : SetServiceCulture / setHostTrace
        CreateServiceInstance --> InvokeService : ServiceUtil.CreateServiceObject
        InvokeService --> ReturnResult : svc.Request
        ReturnResult --> [*]
    }

    CallBSServer --> BSServerSide
    BSServerSide --> BSAgentInvoking
    BSAgentInvoking --> ReturnResult
    ReturnResult --> [*]
```
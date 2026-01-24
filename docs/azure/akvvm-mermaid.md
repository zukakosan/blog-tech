# Azure Key Vault VM 拡張機能 (Windows) フロー図

## 全体アーキテクチャ

```mermaid
flowchart TB
    subgraph Azure["Azure Cloud"]
        KV[("Azure Key Vault<br/>証明書保管")]
    end
    
    subgraph VM["Azure VM"]
        EXT["Key Vault<br/>VM Extension"]
        STORE["Windows<br/>Certificate Store"]
        IIS["IIS"]
    end
    
    CLIENT["Client"]
    
    EXT -->|"1. ポーリング<br/>（定期監視）"| KV
    KV -->|"2. 証明書取得"| EXT
    EXT -->|"3. インストール"| STORE
    STORE -.->|"4. Lifecycle<br/>Notification"| IIS
    IIS -->|"5. 自動バインド"| IIS
    CLIENT -->|"HTTPS"| IIS
```

## ポーリング・更新サイクル

```mermaid
sequenceDiagram
    participant EXT as Key Vault VM Extension
    participant MSI as Managed Identity
    participant KV as Azure Key Vault
    participant STORE as Windows Certificate Store
    participant IIS as IIS

    loop ポーリング間隔ごと
        EXT->>MSI: トークン要求
        MSI-->>EXT: アクセストークン
        EXT->>KV: 監視対象証明書の確認
        KV-->>EXT: 証明書メタデータ
        
        alt 証明書に変更あり
            EXT->>KV: 証明書ダウンロード要求
            KV-->>EXT: 証明書データ
            EXT->>STORE: リーフ証明書インストール
            EXT->>STORE: 中間証明書インストール
            EXT->>STORE: 秘密鍵 ACL 設定
            
            alt linkOnRenewal = true
                EXT->>IIS: Certificate Lifecycle Notification
                IIS->>STORE: 新証明書に再バインド
            end
        end
    end
```

## 拡張機能起動フロー（requireInitialSync = true）

```mermaid
stateDiagram-v2
    [*] --> Starting
    Starting --> Transitioning
    
    Transitioning --> Downloading
    Downloading --> Installing
    Installing --> CheckAll
    
    CheckAll --> Transitioning: 未完了あり
    CheckAll --> Success: 全完了
    
    Downloading --> RetryWait: 失敗
    RetryWait --> Transitioning: リトライ
    RetryWait --> Error: 回数超過
    
    Success --> [*]
    Error --> [*]
```

## 証明書インストール詳細フロー

```mermaid
flowchart TD
    A["証明書ダウンロード完了"] --> B{"証明書の種類を判定"}
    
    B -->|"リーフ証明書"| C["指定ストアにインストール"]
    B -->|"中間 CA 証明書"| D["Intermediate CA ストアにインストール"]
    B -->|"ルート証明書"| E["インストールしない"]
    
    C --> F{"accounts 指定あり?"}
    F -->|"Yes"| G["指定アカウントに読み取り権限付与"]
    F -->|"No"| H["Administrators/SYSTEM のみ"]
    
    G --> I{"keyExportable?"}
    H --> I
    
    I -->|"true"| J["秘密鍵エクスポート可能"]
    I -->|"false"| K["秘密鍵エクスポート不可"]
    
    J --> L{"linkOnRenewal?"}
    K --> L
    
    L -->|"true"| M["Lifecycle Notification 発行"]
    L -->|"false"| N["手動バインディング必要"]
    
    M --> O["インストール完了"]
    N --> O
```

## IIS 証明書バインドフロー

```mermaid
sequenceDiagram
    participant KV as Azure Key Vault
    participant EXT as Key Vault VM Extension
    participant STORE as Windows Certificate Store
    participant HTTP as HTTP.sys
    participant IIS as IIS
    participant SITE as IIS Site Binding

    Note over KV: 証明書が更新される
    
    EXT->>KV: ポーリングで変更検出
    KV-->>EXT: 新しい証明書バージョン
    EXT->>STORE: 新証明書インストール
    
    alt linkOnRenewal = true
        EXT->>HTTP: Certificate Lifecycle Notification 発行
        Note over HTTP: SAN を照合
        HTTP->>IIS: 証明書更新通知
        IIS->>SITE: SSL バインディング更新
        SITE->>STORE: 新証明書にバインド
        Note over SITE: サービス再起動不要
    else linkOnRenewal = false
        Note over IIS: 手動でのバインディング更新が必要
        IIS->>SITE: 管理者が手動で変更
    end
```

## IIS 自動再バインドの詳細フロー

```mermaid
flowchart TB
    subgraph KeyVault["Azure Key Vault"]
        CERT_NEW["新証明書"]
    end
    
    subgraph VMExtension["Key Vault VM Extension"]
        POLL["ポーリング処理"]
        DETECT["変更検出"]
        DOWNLOAD["証明書ダウンロード"]
        INSTALL["証明書インストール"]
        NOTIFY["Lifecycle Notification"]
    end
    
    subgraph WindowsOS["Windows OS"]
        STORE_MY["証明書ストア"]
        STORE_INT["中間証明機関ストア"]
        SCHANNEL["S-channel"]
    end
    
    subgraph IISServer["IIS Server"]
        HTTPAPI["HTTP.sys"]
        W3SVC["W3SVC サービス"]
        BINDING["Site SSL Binding"]
    end
    
    CERT_NEW --> POLL
    POLL --> DETECT
    DETECT -->|"変更あり"| DOWNLOAD
    DOWNLOAD --> INSTALL
    INSTALL --> STORE_MY
    INSTALL --> STORE_INT
    
    INSTALL -->|"linkOnRenewal=true"| NOTIFY
    NOTIFY --> SCHANNEL
    SCHANNEL -->|"SAN 照合"| HTTPAPI
    HTTPAPI --> W3SVC
    W3SVC --> BINDING
    BINDING -->|"新証明書参照"| STORE_MY
```

## IIS バインディング更新の条件分岐

```mermaid
flowchart TD
    START["証明書更新検出"] --> CHECK_LINK{"linkOnRenewal<br/>設定確認"}
    
    CHECK_LINK -->|"true"| AUTO_PATH["自動バインド処理"]
    CHECK_LINK -->|"false"| MANUAL_PATH["手動バインド処理"]
    
    AUTO_PATH --> INSTALL_CERT["新証明書インストール"]
    INSTALL_CERT --> GEN_NOTIFY["Lifecycle Notification 生成"]
    GEN_NOTIFY --> CHECK_SAN{"SAN が一致?"}
    
    CHECK_SAN -->|"一致"| HTTP_UPDATE["HTTP.sys バインディング更新"]
    CHECK_SAN -->|"不一致"| MANUAL_REBIND["手動再バインド必要"]
    
    HTTP_UPDATE --> IIS_AWARE["IIS が新証明書を認識"]
    IIS_AWARE --> NO_RESTART["サービス再起動不要"]
    NO_RESTART --> COMPLETE["自動更新完了"]
    
    MANUAL_PATH --> INSTALL_ONLY["証明書インストールのみ"]
    INSTALL_ONLY --> ADMIN_ACTION["管理者操作が必要"]
    
    ADMIN_ACTION --> OPTION1["IIS Manager で変更"]
    ADMIN_ACTION --> OPTION2["netsh http update sslcert"]
    ADMIN_ACTION --> OPTION3["PowerShell Set-WebBinding"]
    
    OPTION1 --> COMPLETE2["手動更新完了"]
    OPTION2 --> COMPLETE2
    OPTION3 --> COMPLETE2
    
    MANUAL_REBIND --> ADMIN_ACTION
```

## 参考情報

- **linkOnRenewal**: `true` に設定すると、証明書更新時に S-channel バインディングが自動維持される
- **Certificate Lifecycle Notification**: Windows の機能で、IIS 8.5 以降でサポート
- **SAN (Subject Alternative Name)**: 証明書の代替名。IIS は SAN を照合して自動再バインドを実行

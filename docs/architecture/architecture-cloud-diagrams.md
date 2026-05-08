# クラウド移行 HOL - アーキテクチャ図

## 1. 全体ネットワーク構成

```mermaid
flowchart TB
    subgraph onprem["OnPrem VNet 10.0.0.0/16"]
        dc[DC01<br/>AD/DNS]
        dbsrc[DB01<br/>SQL Server]
        appsrc[APP01<br/>IIS/Web]
        onpremvpn[VPN Gateway]
    end

    subgraph hub["Hub VNet 10.10.0.0/16"]
        hubvpn[VPN Gateway]
        fw[Azure Firewall]
        bastion[Azure Bastion]
        dns[DNS Private Resolver]
        law[Log Analytics]
        migrate[Azure Migrate]
    end

    subgraph s1["Spoke1 10.20.0.0/16<br/>Step 5 Rehost"]
        s1web[Web VM]
        s1db[DB VM]
    end

    subgraph s2["Spoke2 10.21.0.0/16<br/>Step 6 DB PaaS"]
        s2web[Web VM]
        s2db[(Azure SQL)]
    end

    subgraph s3["Spoke3 10.22.0.0/16<br/>Step 7 Container"]
        s3app[Container Apps]
        s3db[(Azure SQL)]
    end

    subgraph s4["Spoke4 10.23.0.0/16<br/>Step 8 Full PaaS"]
        s4app[App Service]
        s4db[(Azure SQL)]
    end

    onpremvpn -- "S2S VPN" --> hubvpn
    hubvpn --- fw
    fw --- s1web & s2web & s3app & s4app
    bastion -.-> s1web & s2web
    migrate -.-> appsrc
    hub --- dns
```

## 2. 移行パターン比較フロー

```mermaid
flowchart LR
    SRC[移行元\nAPP01 / DB01] --> A[Step 5 Rehost\nVM + VM]
    SRC --> B[Step 6 DB PaaS 化\nVM + Azure SQL]
    SRC --> C[Step 7 コンテナ化\nContainer Apps + Azure SQL]
    SRC --> D[Step 8 フル PaaS 化\nApp Service + Azure SQL]
```

## 3. ハンズオンの流れ

```mermaid
flowchart TD
    P0[Step 0〓1. 初期環境セットアップ] --> P1[Step 4.1. 移行元確認]
    P1 --> P2[Step 4.2. Azure Arc 登録]
    P2 --> P3[Step 4.3. ハイブリッド管理]
    P3 --> P4[Step 4.4. アセスメント]
    P4 --> P5A[Step 5 Rehost]
    P4 --> P5B[Step 6 DB PaaS 化]
    P4 --> P5C[Step 7 コンテナ化]
    P4 --> P5D[Step 8 フル PaaS 化]
    P5A --> P6[Step 9. 比較・まとめ]
    P5B --> P6
    P5C --> P6
    P5D --> P6
```

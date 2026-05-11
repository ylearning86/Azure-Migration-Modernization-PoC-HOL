# 5. Rehost（Spoke1）

**Azure Migrate（Migration and modernization）** を使い、`vm-app01` と `vm-sql01` を Spoke1 の Azure VM に Lift & Shift 移行します。

> **Nested Hyper-V 版と メイン版の違い**
>
> - **Nested Hyper-V 版**（本手順）: ゲスト VM は実際の Hyper-V VM であるため、**Azure Migrate の Hyper-V 移行**（Site Recovery Provider ベース）を使用します。これは実際のオンプレ移行と同じフローです。
> - **メイン版**: 疑似オンプレ VM が Azure IaaS 上で動作しているため、Azure Migrate のモビリティエージェントが使えず、代替として Azure Site Recovery（Azure-to-Azure レプリケーション）を使用します。メイン版の手順は本ドキュメント末尾の「参考」セクションを参照してください。

## 目的

- Azure Migrate による Hyper-V VM のレプリケーション・移行を体験する
- テスト移行 → 本番移行（カットオーバー）の一連のフローを理解する
- Rehost（Lift & Shift）パターンのメリット・デメリットを理解する

## 前提条件

- [`4.4-cloud-assessment.md`](./4.4-cloud-assessment.md) の評価が完了している
- `rg-spoke1`・`vnet-spoke1`（snet-web / snet-db）が作成されている
- Hub-Spoke VNet ピアリングが接続済みである

## 移行先構成

| コンポーネント | 移行元 | 移行先 |
|---|---|---|
| Web サーバー | vm-app01（192.168.100.11） | vm-spoke1-web（snet-web / 10.20.x.x） |
| DB サーバー | vm-sql01（192.168.100.12） | vm-spoke1-sql（snet-db / 10.20.x.x） |

## 特徴

- **メリット**: アプリやOS への変更が不要。最も速い移行パターン
- **デメリット**: VM 運用が継続するため、運用負荷は高いまま

---

<details>
<summary>（任意）PsPing のインストール</summary>

移行中のサービス断を確認するため、DC01 に [PsPing](https://learn.microsoft.com/ja-jp/sysinternals/downloads/psping?wt.mc_id=MVP_479930) をインストールします。PsPing は TCP ポート単位の疎通確認ができるため、Windows Firewall で ICMP がブロックされていても使えます。

### インストール手順

1. Azure Bastion 経由で **vm-ad01**（192.168.100.10）にホスト VM 経由で接続（PowerShell Direct）
2. 管理者 PowerShell で以下を実行:

   ```powershell
   # PSTools をダウンロード・展開
   [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
   Invoke-WebRequest -Uri "https://download.sysinternals.com/files/PSTools.zip" -OutFile "$env:TEMP\PSTools.zip" -UseBasicParsing
   Expand-Archive -Path "$env:TEMP\PSTools.zip" -DestinationPath "C:\Tools" -Force

   # PATH に追加
   $env:Path += ";C:\Tools"
   [Environment]::SetEnvironmentVariable("Path", "$([Environment]::GetEnvironmentVariable('Path','Machine'));C:\Tools", "Machine")
   ```

### 使い方

移行テスト時に DC01 のコマンドプロンプトまたは PowerShell で実行します:

```cmd
:: APP01 の HTTP (ポート 80) を継続監視（Ctrl+C で停止）
psping64 -accepteula -t 192.168.100.11:80

:: SQL01 の SQL Server (ポート 1433) を継続監視
psping64 -accepteula -t 192.168.100.12:1433
```

移行中にサービスが停止すると応答が途切れるため、ダウンタイムの開始・終了時刻を確認できます。

> **補足**: PsPing は [Sysinternals PSTools](https://learn.microsoft.com/ja-jp/sysinternals/downloads/psping) に含まれるスタンドアロン exe で、コマンドプロンプト・PowerShell の両方で使用可能です。

</details>

---

## Nested Hyper-V 版の手順

> 以下は Nested Hyper-V 環境で **Azure Migrate の Hyper-V 移行（Migration and modernization）** を使う手順です。メイン版（ASR Azure-to-Azure）の手順は後半の「参考」セクションを参照してください。

### Step 1: Site Recovery Provider のセットアップ

Azure Migrate の Hyper-V 移行では、**Hyper-V ホストに Site Recovery Provider をインストール** してレプリケーションを管理します。

#### 1-1. リソースの作成と Provider のダウンロード

1. Azure Portal → **Azure Migrate** → `migr-project` → **Execute** → **Migrations** → **Start execution**
2. **Specify intent**:
   - What do you want to migrate → **Servers or virtual machines (VMs)**
   - Where do you want to migrate to → **Azure VM**
   - How will you select workloads → **From an assessment** または **From all inventory**
3. **Discovery method** → Hyper-V アプライアンスを選択 → **Next**
4. **Target region**: `Japan East` → **Confirm** → **Create resources**（Recovery Services vault が自動作成される）
5. **Prepare Hyper-V host servers** で以下をダウンロード:
   - **AzureSiteRecoveryProvider.exe**（Provider インストーラー）
   - **Registration key**（.VaultCredentials ファイル）

#### 1-2. ファイルの転送

ダウンロードした 2 ファイルを Hyper-V ホスト VM（`vm-onprem-nested-hv01`）に転送します。Blob Storage 経由が確実です。

```powershell
# ローカル PC から Blob にアップロード（既存の Storage Account を使用）
az storage blob upload --account-name <storage-account> --container-name upload --file AzureSiteRecoveryProvider.exe --name AzureSiteRecoveryProvider.exe --auth-mode login --overwrite -o none
az storage blob upload --account-name <storage-account> --container-name upload --file <VaultCredentials ファイル> --name registration-key.VaultCredentials --auth-mode login --overwrite -o none

# SAS URL を生成
$expiry = (Get-Date).ToUniversalTime().AddHours(2).ToString("yyyy-MM-ddTHH:mmZ")
az storage blob generate-sas --account-name <storage-account> --container-name upload --name AzureSiteRecoveryProvider.exe --permissions r --expiry $expiry --auth-mode login --as-user --full-uri -o tsv
az storage blob generate-sas --account-name <storage-account> --container-name upload --name registration-key.VaultCredentials --permissions r --expiry $expiry --auth-mode login --as-user --full-uri -o tsv
```

ホスト VM 上で SAS URL を使ってダウンロード:

```powershell
New-Item -Path C:\ASRProvider -ItemType Directory -Force
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest -Uri '<Provider SAS URL>' -OutFile C:\ASRProvider\AzureSiteRecoveryProvider.exe -UseBasicParsing
Invoke-WebRequest -Uri '<VaultCredentials SAS URL>' -OutFile C:\ASRProvider\registration-key.VaultCredentials -UseBasicParsing
```

#### 1-3. Provider のインストールと登録

ホスト VM の管理者 PowerShell で実行:

```powershell
# 展開
& C:\ASRProvider\AzureSiteRecoveryProvider.exe /q /x:"C:\ASRProvider\Extracted"

# インストール
cd C:\ASRProvider\Extracted
.\setupdr.exe /i

# 登録
& "C:\Program Files\Microsoft Azure Site Recovery Provider\DRConfigurator.exe" /r /Credentials "C:\ASRProvider\registration-key.VaultCredentials"
```

#### 1-4. 登録の完了

1. Azure Portal の Provider セットアップ画面に戻る
2. **Finalize registration** をクリック
3. 最大 15 分で登録が反映される

---

### Step 2: レプリケーションの開始

1. **Execute** → **Migrations** → **Start execution** で、Provider セットアップ済みの画面に進む
2. **Workloads** タブ: `vm-app01` と `vm-sql01` を選択 → **Next**
3. **Target settings**:

   | 項目 | 値 |
   | --- | --- |
   | サブスクリプション | 使用中のサブスクリプション |
   | ターゲットリージョン | Japan East |
   | リソースグループ | `rg-spoke1` |
   | 仮想ネットワーク | `vnet-spoke1` |
   | サブネット (vm-app01) | `snet-web` |
   | サブネット (vm-sql01) | `snet-db` |
   | Cache storage account | 既定（自動作成） |

4. **Compute**: VM サイズを確認（評価の推奨サイズまたは手動選択）
5. **Tags**: `Environment=PoC`, `Project=Migration-Handson`, `SecurityControl=Ignore`
6. **Review and start execution** → レプリケーション開始

#### レプリケーション状態の監視

**Execute** → **Migrations** で進行状況を確認:

| ステージ | 説明 | 次のアクション |
| --- | --- | --- |
| **Preparation** | 初期レプリケーション（フルコピー）進行中 | 完了を待つ（数十分〜数時間） |
| **Testing** | 差分レプリケーション中、テスト移行が可能 | テスト移行を実行 |
| **Completion** | テスト完了後、本番移行が可能 | カットオーバーを実行 |

---

### Step 3: テスト移行

テスト移行はソース VM に影響を与えません。移行先で VM が正常に起動するかを検証します。

#### 3-1. テスト移行の実行

vm-app01 と vm-sql01 それぞれで実行:

1. **Execute** → **Migrations** → ワークロード名をクリック
2. **Testing** ドロップダウン → **Start test migration**
3. 設定:

   | 項目 | 値 |
   | --- | --- |
   | Virtual network | `vnet-spoke1` |

   > テスト移行では VNet のみ選択します。サブネットは Step 2 で設定した値が使用されます。

4. **Test migration** をクリック（完了まで数分〜十数分）

#### 3-2. テスト VM の確認

テスト移行が完了すると、`rg-spoke1` に `-test` サフィックス付きの VM が作成されます。

| 項目 | 確認方法 |
| --- | --- |
| VM の状態 | Azure Portal で `-test` VM が「実行中」 |
| IIS | Web テスト VM に Bastion 接続 → `iisreset /status` |
| SQL Server | SQL テスト VM に Bastion 接続 → `Get-Service MSSQLSERVER` |
| Parts Unlimited | Web テスト VM 上のブラウザで `http://localhost` にアクセス |

> **注意**: テスト VM の Web.config 接続文字列はまだ旧 IP（`192.168.100.12`）を指しているため、Parts Unlimited は DB 接続エラーになります。これはテスト移行の段階では想定内です。動作を確認したい場合は、テスト Web VM 上で Web.config を一時的にテスト SQL VM の IP に変更してください。

#### 3-3. テスト移行のクリーンアップ

確認後、テスト VM を削除します:

1. **Execute** → **Migrations** → ワークロード名をクリック
2. **Testing** ドロップダウン → **Clean up test migration**
3. 確認して削除

> vm-app01 / vm-sql01 の両方でクリーンアップしてください。

---

### Step 4: 本番移行（カットオーバー）

#### 4-1. ダウンタイム監視の開始（任意）

本番移行を開始する前に、vm-ad01 で PsPing / HTTP 監視を開始しておくとダウンタイムを計測できます（前述の PsPing セクションを参照）。

#### 4-2. 移行の実行

vm-app01 と vm-sql01 それぞれで実行:

1. **Execute** → **Migrations** → ワークロード名をクリック
2. **Completion** ドロップダウン → **Migrate**
3. **Shut down virtual machines and perform a planned migration with no data loss** → **Yes**
4. **Migrate** をクリック

> ソース VM がシャットダウンされ、最終差分同期後に Azure VM が作成されます。

#### 4-3. 移行の完了

1. 移行ジョブが完了したら、ワークロードの drill-down ページを開く
2. **Completion** → **Complete migration** をクリック
3. レプリケーションが停止され、移行リソースがクリーンアップされる

---

### Step 5: 移行後の設定と動作確認

#### 5-1. 移行先 VM の IP 確認

Azure Portal で `rg-spoke1` 内の移行先 VM のプライベート IP を確認:

| VM | サブネット | 新 IP（例） |
| --- | --- | --- |
| vm-app01 | snet-web | 10.20.x.x |
| vm-sql01 | snet-db | 10.20.x.x |

#### 5-2. 接続文字列の更新

移行先 vm-app01 に Bastion 接続し、管理者 PowerShell で実行:

```powershell
$webConfig = 'C:\inetpub\PartsUnlimited\Web.config'
[xml]$xml = Get-Content $webConfig
$conn = $xml.configuration.connectionStrings.add |
    Where-Object { $_.name -eq 'DefaultConnectionString' }
$conn.connectionString = $conn.connectionString -replace '192\.168\.100\.12', '<新SQL_IP>'
$xml.Save($webConfig)
iisreset
```

#### 5-3. DNS レコードの更新

vm-ad01（ホスト VM 経由で PowerShell Direct）で実行:

```powershell
$pw = ConvertTo-SecureString 'P@ssW0rd1234!' -AsPlainText -Force
$cred = New-Object PSCredential('contoso\Administrator', $pw)
Invoke-Command -VMName vm-ad01 -Credential $cred -ScriptBlock {
    Remove-DnsServerResourceRecord -ZoneName contoso.local -Name vm-app01 -RRType A -Force -ErrorAction SilentlyContinue
    Add-DnsServerResourceRecordA -ZoneName contoso.local -Name vm-app01 -IPv4Address '<新Web_IP>'
    Remove-DnsServerResourceRecord -ZoneName contoso.local -Name vm-sql01 -RRType A -Force -ErrorAction SilentlyContinue
    Add-DnsServerResourceRecordA -ZoneName contoso.local -Name vm-sql01 -IPv4Address '<新SQL_IP>'
}
```

#### 5-4. 動作確認

| 項目 | 確認方法 |
| --- | --- |
| VM の状態 | Azure Portal で Spoke1 の VM が「実行中」 |
| IIS | Web VM に Bastion 接続 → `iisreset /status` |
| SQL Server | SQL VM に Bastion 接続 → `Get-Service MSSQLSERVER` |
| Parts Unlimited | Web VM 上のブラウザで `http://localhost` にアクセス |
| ネットワーク疎通 | Web VM → SQL VM: `Test-NetConnection <新SQL_IP> -Port 1433` |

---

## 完了確認（Nested Hyper-V 版）

- [ ] rg-spoke1 に移行された VM が稼働している
- [ ] Parts Unlimited がブラウザで表示できる
- [ ] 移行元の vm-app01 / vm-sql01 が停止されている
- [ ] Azure Migrate で Complete migration が実行済み

---

<details>
<summary>参考: メイン版の手順（Azure Site Recovery / Azure-to-Azure）</summary>

> 以下はメイン版（疑似オンプレ VM が Azure IaaS 上にある場合）の手順です。Nested Hyper-V 環境では前述の Azure Migrate Hyper-V 移行を使用してください。

</details>

<details>
<summary>Step 1: Recovery Services コンテナーの作成</summary>

Azure-to-Azure Site Recovery では、Recovery Services コンテナーがレプリケーションの管理拠点になります。

### 1-1. コンテナーの作成

Azure Portal または CLI で Recovery Services コンテナーを作成します。

| 項目 | 値 |
| --- | --- |
| リソース グループ | rg-spoke1 |
| コンテナー名 | rsv-spoke1 |
| リージョン | Japan East（ソース VM と同じリージョン） |

<details>
<summary>CLI で作成する場合</summary>

```bash
# Recovery Services コンテナーの作成
az backup vault create \
  --resource-group rg-spoke1 \
  --name rsv-spoke1 \
  --location japaneast \
  --tags Environment=PoC Project=Migration-Handson SecurityControl=Ignore
```

</details>

### 1-2. Site Recovery の有効化確認

1. Azure Portal で **rsv-spoke1** を開く
2. 左メニューの **[Site Recovery]** → **[Azure 仮想マシン]** セクションが表示されることを確認

> **参考**: [Azure VM の Azure へのディザスター リカバリーを設定する](https://learn.microsoft.com/ja-jp/azure/site-recovery/azure-to-azure-tutorial-enable-replication?wt.mc_id=MVP_479930)

</details>

---

<details>
<summary>Step 2: レプリケーションの有効化</summary>

ソース VM（APP01 / DB01）から Spoke1 VNet へのレプリケーションを設定します。同一リージョン内のレプリケーションとなります。

### 2-1. APP01 のレプリケーション

1. Azure Portal で **rsv-spoke1** → **[Site Recovery]** → **[Azure 仮想マシン]** → **[+ レプリケーションを有効にする]**
2. **[ソース]** タブ:

   | 項目 | 値 |
   | --- | --- |
   | リージョン | Japan East |
   | サブスクリプション | 使用中のサブスクリプション |
   | リソース グループ | rg-onprem |
   | 仮想マシンのデプロイ モデル | Resource Manager |

3. **[仮想マシン]** タブで **vm-onprem-web**（APP01）を選択
4. **[レプリケーションの設定]** タブ:

   | 項目 | 値 |
   | --- | --- |
   | ターゲット リソース グループ | rg-spoke1 |
   | ターゲット仮想ネットワーク | vnet-spoke1 |
   | ターゲット サブネット | snet-web |
   | ストレージ | （自動作成されるキャッシュ ストレージ アカウント） |
   | レプリケーション ポリシー | （既定のポリシーまたは新規作成） |

5. **[レプリケーションを有効にする]** をクリック

### 2-2. DB01 のレプリケーション

同じ手順で DB01 用のレプリケーションを追加します。

1. **rsv-spoke1** → **[Site Recovery]** → **[Azure 仮想マシン]** → **[+ レプリケーションを有効にする]**
2. **[ソース]** タブ: APP01 と同じ設定（rg-onprem）
3. **[仮想マシン]** タブで **vm-onprem-sql**（DB01）を選択
4. **[レプリケーションの設定]** タブ:

   | 項目 | 値 |
   | --- | --- |
   | ターゲット リソース グループ | rg-spoke1 |
   | ターゲット仮想ネットワーク | vnet-spoke1 |
   | ターゲット サブネット | snet-db |

5. **[レプリケーションを有効にする]** をクリック

> **ポイント**: Azure-to-Azure レプリケーションでは、モビリティサービス拡張機能がソース VM に自動的にインストールされます。レプリケーションアプライアンスは不要です。

### レプリケーション状態の確認

**rsv-spoke1** → **[レプリケートされたアイテム]** で各 VM のステータスを確認します。

| ステータス | 説明 |
| --- | --- |
| 初期レプリケーション中 | 最初のフルコピーが進行中 |
| 保護済み | 差分レプリケーションが継続中 — テスト フェールオーバーが可能 |
| 重大 | エラーが発生、対処が必要 |

**[保護済み]** になったら次のステップへ進みます（数十分〜数時間）。

</details>

---

<details>
<summary>Step 3: テスト フェールオーバー</summary>

本番カットオーバー前に、テスト フェールオーバーでアプリケーションの動作を確認します。テスト フェールオーバーはソース VM に影響を与えません。

### 3-1. テスト フェールオーバーの実行

1. **rsv-spoke1** → **[レプリケートされたアイテム]** で対象 VM を選択
2. **[テスト フェールオーバー]** をクリック
3. 以下を設定:

   | 項目 | 値 |
   | --- | --- |
   | 復旧ポイント | 最新の処理済み（低 RTO） |
   | Azure 仮想ネットワーク | vnet-spoke1 |

4. **[OK]** をクリック

> **注意**: APP01 と DB01 の両方でテスト フェールオーバーを実行してください。

### 3-2. テスト フェールオーバー後の確認

テスト フェールオーバーが完了すると、rg-spoke1 に `-test` サフィックスの付いた VM が作成されます。

| 項目 | 確認方法 |
| --- | --- |
| VM の状態 | Azure Portal で `-test` VM が「実行中」 |
| IIS | テスト VM に Bastion 接続 → `iisreset /status` |
| SQL Server | テスト VM に Bastion 接続 → SSMS で DB 接続確認 |
| Parts Unlimited | ブラウザで `http://<テスト VM の IP>` にアクセス |

### 3-3. テスト フェールオーバーのクリーンアップ

確認が終わったら、テスト VM を削除します。

1. **[レプリケートされたアイテム]** で対象 VM を選択
2. **[テスト フェールオーバーのクリーンアップ]** をクリック
3. メモを入力 → **[テストが完了しました。テスト フェールオーバー仮想マシンを削除してください。]** にチェック → **[OK]**

</details>

---

<details>
<summary>Step 4: カットオーバー（本番フェールオーバー）</summary>

テスト フェールオーバーで問題がなければ、本番フェールオーバーを実行します。

> **重要**: フェールオーバーを実行すると、ソース VM が停止されます。事前に計画停止時間を確保してください。

### 4-1. ソース VM のサービス停止

1. APP01: IIS の停止
2. DB01: SQL Server サービスの停止

### 4-2. フェールオーバーの実行

1. **rsv-spoke1** → **[レプリケートされたアイテム]** で対象 VM を選択
2. **[フェールオーバー]** をクリック
3. 以下を設定:

   | 項目 | 値 |
   | --- | --- |
   | 復旧ポイント | 最新の処理済み（低 RTO） |
   | フェールオーバーを開始する前にマシンをシャットダウンします | はい |

4. **[OK]** をクリック

> **注意**: APP01 と DB01 の両方でフェールオーバーを実行してください。

### 4-3. フェールオーバーのコミット

フェールオーバーが完了したら、**コミット** して確定します。

1. **[レプリケートされたアイテム]** で対象 VM を選択
2. **[コミット]** をクリック

> **補足**: コミット後、フェールオーバー先の VM が正式な稼働環境になります。コミットしないとフェールオーバーは未確定のままで、別の復旧ポイントに切り替えることも可能です。

### 4-4. フェールオーバー後の VM 名変更（任意）

フェールオーバー後の VM はソース VM と同じ名前で作成されます。HOL の命名規則に合わせたい場合は、Azure Portal またはCLI で VM 名を変更してください。

</details>

---

<details>
<summary>Step 5: 動作確認</summary>

フェールオーバー後、移行先 VM で Parts Unlimited が正常に動作するか確認します。

### 確認項目

| 項目 | 確認方法 |
| --- | --- |
| VM の状態 | Azure Portal で Spoke1 の VM が「実行中」 |
| IIS | Web VM に Bastion 接続 → `iisreset /status` |
| SQL Server | SQL VM に Bastion 接続 → SSMS で DB 接続確認 |
| Parts Unlimited | ブラウザで `http://<Web VM の IP>` にアクセス |

### ネットワーク構成の確認

移行先 VM が Hub-Spoke ネットワーク経由で通信できるよう、以下を確認します:

- Web VM → SQL VM 間の SQL Server 接続文字列を更新（IP が変わった場合）
- NSG で必要なポート（80, 1433 等）が開放されているか確認
- DNS レコードの更新（必要に応じて）

### レプリケーションの無効化

動作確認が完了したら、レプリケーションを無効化します。

1. **rsv-spoke1** → **[レプリケートされたアイテム]** で対象 VM を選択
2. **[レプリケーションの無効化]** をクリック

</details>

---

<details>
<summary>トラブルシューティング</summary>

### レプリケーションの有効化が失敗する

- ソース VM が実行中であることを確認
- ソース VM に Azure VM エージェント（waagent）がインストールされていることを確認
- ターゲットのリソース グループ / VNet / サブネットが存在していることを確認

### テスト フェールオーバー後の VM にアクセスできない

- NSG ルールで Bastion からの RDP（3389）が許可されているか確認
- テスト VM が正しい VNet / サブネットに接続されているか確認
- テスト VM のパブリック IP が必要な場合は手動で付与する

### フェールオーバー後に Parts Unlimited が動作しない

- Web VM と SQL VM の間のネットワーク疎通を確認（`Test-NetConnection` / PsPing）
- 接続文字列が移行先の IP / ホスト名に更新されているか確認
- IIS / SQL Server のサービスが起動しているか確認

> **参考**: [Azure VM のディザスター リカバリーのトラブルシューティング](https://learn.microsoft.com/ja-jp/azure/site-recovery/azure-to-azure-troubleshoot-errors?wt.mc_id=MVP_479930)

</details>

---

<details>
<summary>移行チェックリスト — Rehost 時の注意事項</summary>

Azure へのリホスト（Lift & Shift）移行で見落としやすい項目を体系的にまとめます。

### 1. ネットワーク / 接続性

| # | 確認項目 | 本 HOL での対処 |
|---|---------|---------------|
| 1-1 | **IP アドレスの変更** — 移行先 VM の IP が変わる | Web.config 接続文字列を新 IP に更新 |
| 1-2 | **DNS レコードの更新** — A レコードが旧 IP を指したまま | vm-ad01 で DNS A レコードを更新（APP01 / SQL01） |
| 1-3 | **NSG ルール** — 移行先サブネットでポートが開放されているか | snet-web: 80/443, snet-db: 1433 を確認 |
| 1-4 | **ファイアウォール（Azure Firewall / OS）** — 通信経路が許可されているか | Hub-Spoke 構成の場合 UDR / Firewall ルールを確認 |
| 1-5 | **VNet ピアリング / VPN** — 移行先 VNet から他のリソースへの疎通 | vnet-spoke1 ↔ vnet-hub ピアリング済みを確認 |

### 2. アプリケーション

| # | 確認項目 | 本 HOL での対処 |
|---|---------|---------------|
| 2-1 | **接続文字列** — DB の IP / ホスト名がハードコードされていないか | `Web.config` の `DefaultConnectionString` を更新 |
| 2-2 | **環境依存の設定ファイル** — ログパス、一時ファイルパス等 | IIS サイトパス `C:\inetpub\PartsUnlimited` がそのまま移行される |
| 2-3 | **サービスの自動起動** — IIS / SQL Server が OS 起動時に開始するか | `Get-Service W3SVC, MSSQLSERVER` で StartType を確認 |
| 2-4 | **IIS バインド** — ホスト名バインドが旧ホスト名になっていないか | `Get-WebBinding` で確認。IP `*` バインドなら問題なし |
| 2-5 | **依存サービスの確認** — AD 認証、NTP、外部 API 等 | Parts Unlimited は SQL 認証のため AD 依存なし |

### 3. OS / ドライバー

| # | 確認項目 | 本 HOL での対処 |
|---|---------|---------------|
| 3-1 | **Azure VM エージェント** — waagent がインストールされ動作しているか | Azure Migrate が移行時に自動インストール |
| 3-2 | **Windows ライセンスのアクティベーション** — KMS / AVMA が Azure で有効か | [Azure でのアクティベーションのトラブルシューティング](https://learn.microsoft.com/troubleshoot/azure/virtual-machines/troubleshoot-activation-problems) を参照 |
| 3-3 | **ディスクのドライブレター** — OS ディスク以外のデータディスクの割り当て | 移行後にディスク管理で確認 |
| 3-4 | **タイムゾーン / NTP** — Azure VM の既定は UTC | `Set-TimeZone -Id "Tokyo Standard Time"` で変更可能 |
| 3-5 | **RDP / Bastion アクセス** — 移行先 VM に管理アクセスできるか | Bastion 経由で接続確認 |

### 4. セキュリティ

| # | 確認項目 | 本 HOL での対処 |
|---|---------|---------------|
| 4-1 | **管理者パスワード** — 移行後も変更されていないか | 既存パスワードが引き継がれる |
| 4-2 | **Azure Disk Encryption** — 保存時暗号化が必要か | PoC のため既定の SSE（プラットフォームマネージドキー）で十分 |
| 4-3 | **Just-in-Time VM アクセス** — 管理ポートの公開制限 | Bastion 経由のため直接 RDP は不要 |
| 4-4 | **Azure Update Manager** — OS / アプリのパッチ管理 | 移行後に有効化を検討 |
| 4-5 | **NSG フローログ** — ネットワークトラフィックの監査 | 必要に応じて有効化 |

### 5. 移行後の運用（Post-migration）

| # | 確認項目 | 本 HOL での対処 |
|---|---------|---------------|
| 5-1 | **Azure Backup** — VM のバックアップを構成 | [クイックスタート: VM バックアップ](https://learn.microsoft.com/azure/backup/quick-backup-vm-portal) |
| 5-2 | **監視** — Azure Monitor / AMA でメトリクスとログを収集 | Step 4.3 で AMA + DCR 構成済み（移行先 VM にも適用が必要） |
| 5-3 | **コスト管理** — 適切な VM サイズか、予約インスタンスの検討 | Azure Migrate 評価の推奨サイズを参考に |
| 5-4 | **タグ付け** — 移行先 VM にプロジェクトタグが付与されているか | `Environment=PoC`, `Project=Migration-Handson`, `SecurityControl=Ignore` |
| 5-5 | **移行元の停止 / 削除** — ソース VM をデコミッションする | 移行確認後にソース VM を停止 |
| 5-6 | **SQL Server IaaS Agent 拡張** — SQL VM の場合に登録推奨 | [SQL IaaS Agent Extension](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-server-iaas-agent-extension-automate-management) |
| 5-7 | **DR 構成** — 移行先 VM の冗長化が必要か | 必要に応じて Site Recovery で別リージョンへレプリケーション |

### 本 HOL での移行後タスク（最低限）

```powershell
# === 移行先 vm-app01（Web）で実行 ===
# 1. 接続文字列を更新
$webConfig = 'C:\inetpub\PartsUnlimited\Web.config'
[xml]$xml = Get-Content $webConfig
$conn = $xml.configuration.connectionStrings.add |
    Where-Object { $_.name -eq 'DefaultConnectionString' }
$conn.connectionString = $conn.connectionString -replace '192\.168\.100\.12', '<新SQL_IP>'
$xml.Save($webConfig)
iisreset

# === vm-ad01（DC）で実行 ===
# 2. DNS A レコード更新
Remove-DnsServerResourceRecord -ZoneName contoso.local -Name vm-app01 -RRType A -Force -ErrorAction SilentlyContinue
Add-DnsServerResourceRecordA -ZoneName contoso.local -Name vm-app01 -IPv4Address <新Web_IP>
Remove-DnsServerResourceRecord -ZoneName contoso.local -Name vm-sql01 -RRType A -Force -ErrorAction SilentlyContinue
Add-DnsServerResourceRecordA -ZoneName contoso.local -Name vm-sql01 -IPv4Address <新SQL_IP>
```

> **参考**: [Azure VM への移行後のベスト プラクティス](https://learn.microsoft.com/azure/migrate/tutorial-migrate-hyper-v?view=migrate#post-migration-best-practices) | [SQL Server on Azure VM パフォーマンスのベスト プラクティス](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/performance-guidelines-best-practices-checklist)

</details>

---

## 完了確認

- [ ] rg-spoke1 にフェールオーバーした VM が稼働している
- [ ] Parts Unlimited がブラウザで表示できる
- [ ] 移行元の APP01 / DB01 が停止されている
- [ ] レプリケーションが無効化されている

---

<details>
<summary>参考: エージェントベース方式の手順（Azure Migrate / Migration and modernization）</summary>

> **注意**: 以下は Azure Migrate の Migration and modernization（エージェントベース方式）を使用する場合の手順です。ソース VM が Azure IaaS 上にある場合はモビリティエージェントのインストールが製品仕様上スキップされるため、本ハンズオン環境では実行できません。将来的に Hyper-V nested virtualization 環境を用意した場合に参照してください。

以下の手順はエージェントベース方式での一連の流れです:

1. **レプリケーションアプライアンス VM の作成**: rg-onprem に vm-onprem-repl（Windows Server 2022 / D16s_v3 / OS ディスク 1TB）を作成
2. **Azure Migrate で移行の意図を指定**: Azure Migrate プロジェクトで「物理またはその他」を選択
3. **レプリケーションアプライアンスのセットアップ**: DRInstaller.ps1 を実行し、Configuration Manager でアプライアンスを登録
4. **レプリケーションの有効化**: APP01 / DB01 を別々に設定（ターゲット サブネットが異なるため）
5. **テスト移行**: テスト VM で Parts Unlimited の動作確認
6. **カットオーバー**: 最終差分同期後に本番移行を実行
7. **動作確認**: 移行先 VM で Parts Unlimited が正常動作することを確認

詳細手順は [Azure Migrate ドキュメント](https://learn.microsoft.com/ja-jp/azure/migrate/tutorial-migrate-physical-virtual-machines?wt.mc_id=MVP_479930) を参照してください。

</details>

## 次のステップ

➡ [Step 6: DB PaaS 化](./6-cloud-db-paas.md)
➡ [Step 9: 比較・まとめ](./9-cloud-compare.md)

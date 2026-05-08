# Azure Migration & Modernization PoC ハンズオンラボ

オンプレミス環境を模した Windows Server、SQL Server、.NET Framework ワークロードの Web 3 層アプリケーションを題材に、**Azure への移行**と**モダナイゼーション**を段階的に体験できるハンズオンラボです。  
移行対象アプリには Microsoft 公式サンプルの **[Parts Unlimited](https://github.com/Microsoft/PartsUnlimitedE2E)**（ASP.NET MVC / .NET Framework）を使用します。  
Azure 上の **Nested Hyper-V** で疑似オンプレ環境（`vm-ad01` / `vm-app01` / `vm-sql01`）を構築し、Azure Arc・Azure Migrate・各種 PaaS を使いながら、複数の移行パターンを比較できます。

---

## 概要

このラボでは、以下の一連の流れを体験できます。

- Nested Hyper-V による疑似オンプレ環境の構築とアプリ動作確認
- Azure Arc によるハイブリッド管理
- Azure Migrate によるアセスメント（Hyper-V 検出）
- 4 つの移行/モダナイズ パターンの比較
  - Rehost
  - DB PaaS 化
  - コンテナ化
  - フル PaaS 化

---

## 対象者

- Azure への移行を提案・設計するパートナー/SE
- オンプレミス ワークロードのクラウド移行を検討するお客様
- IaaS から PaaS/コンテナまでの比較検討をしたい技術者
- Azure Arc / Azure Migrate を実際の流れに沿って学びたい方

---

## このラボで学べること

| テーマ | 学べる内容 |
|---|---|
| 移行元の理解 | 既存アプリ/DB/AD を含む典型的な構成の把握 |
| ハイブリッド管理 | Azure Arc / Policy / Monitor / Defender の活用 |
| アセスメント | Azure Migrate による移行前評価（Hyper-V 検出） |
| 移行パターン比較 | VM 維持、DB のみ PaaS、コンテナ化、フル PaaS の違い |
| モダナイゼーション | .NET アプリの段階的な改善アプローチ |

---

## ラボ シナリオ

Nested Hyper-V 上のゲスト VM（`vm-ad01` / `vm-app01` / `vm-sql01`）を移行元として、クラウド基盤を構築し、移行評価とモダナイズ比較を進めます。  
疑似オンプレ環境は Nested Hyper-V で構築します。設計・構築手順の詳細は [`docs/architecture/architecture-nested-hyperv.md`](./docs/architecture/architecture-nested-hyperv.md) を参照してください。

| Step | ドキュメント | 内容 | 所要時間目安 |
|---|---|---|---|
| 0 | [`architecture-nested-hyperv.md`](./docs/architecture/architecture-nested-hyperv.md) §2〜3 | 前提条件確認 / Windows Server VHD の入手 | 事前準備 |
| 1 | 同 §4 (4.1〜4.8) | 疑似オンプレ環境の構築（Bicep デプロイ → VHD アップロード → VM 構築 → ドメイン参加） | 2〜3 時間 |
| 1+ | 同 備考: SQL Server + [`1.3`](./docs/handson/1.3-onprem-parts-unlimited.md) | SQL Server インストール + Parts Unlimited セットアップ | 30〜60 分 |
| 2 | [`1.5-cloud-deploy.md`](./docs/handson/1.5-cloud-deploy.md) | 移行先クラウド基盤（Hub & Spoke）をデプロイ | 45〜60 分 |
| 3 | [`architecture-nested-hyperv.md`](./docs/architecture/architecture-nested-hyperv.md) §5〜6 | VPN 接続 + ハイブリッド DNS を構成 | 45〜60 分 |
| 4.1 | [`2.1-cloud-explore-onprem.md`](./docs/handson/2.1-cloud-explore-onprem.md) | 移行元環境の現状確認 | 10〜15 分 |
| 4.2 | [`2.2-cloud-arc-onboard.md`](./docs/handson/2.2-cloud-arc-onboard.md) | Azure Arc 登録（ゲスト VM を Arc 対応） | 15〜20 分 |
| 4.3 | [`2.3-cloud-hybrid-mgmt.md`](./docs/handson/2.3-cloud-hybrid-mgmt.md) | ハイブリッド管理を体験 | 15〜20 分 |
| 4.4 | [`2.4-cloud-assessment.md`](./docs/handson/2.4-cloud-assessment.md) | Azure Migrate で評価（Hyper-V 検出） | 30〜45 分 |
| 5 | [`2.5.1-cloud-rehost.md`](./docs/handson/2.5.1-cloud-rehost.md) | Rehost — Azure Migrate Hyper-V 移行で Lift & Shift | 30〜45 分 |
| 6 | [`2.5.2-cloud-db-paas.md`](./docs/handson/2.5.2-cloud-db-paas.md) | DB PaaS 化を実施 | 30〜45 分 |
| 7 | [`2.5.3-cloud-containerize.md`](./docs/handson/2.5.3-cloud-containerize.md) | コンテナ化を実施 | 45〜60 分 |
| 8 | [`2.5.4-cloud-full-paas.md`](./docs/handson/2.5.4-cloud-full-paas.md) | フル PaaS 化を実施 | 30〜45 分 |
| 9 | [`2.6-cloud-compare.md`](./docs/handson/2.6-cloud-compare.md) | 結果の比較とまとめ | 10〜15 分 |
| 10 | [`2.7-cloud-cleanup.md`](./docs/handson/2.7-cloud-cleanup.md) | リソースのクリーンアップ | 5〜10 分 |

---

## アーキテクチャ概要

- **移行元**: Nested Hyper-V 上の `vm-ad01` / `vm-app01` / `vm-sql01` による疑似オンプレ 3 層構成
- **移行先**: Hub & Spoke をベースにした Azure 環境
- **比較対象**:
  - Spoke1: Rehost
  - Spoke2: DB PaaS 化
  - Spoke3: コンテナ化
  - Spoke4: フル PaaS 化

詳細は以下を参照してください。

- [`docs/architecture/architecture-nested-hyperv.md`](./docs/architecture/architecture-nested-hyperv.md) — **Nested Hyper-V 疑似オンプレ環境の設計・構築手順**
- [`docs/architecture/architecture-cloud-design.md`](./docs/architecture/architecture-cloud-design.md) — Hub & Spoke / 移行パターン設計
- [`docs/architecture/architecture-cloud-diagrams.md`](./docs/architecture/architecture-cloud-diagrams.md) — クラウド構成図

---

## 前提条件

- Azure サブスクリプション（Contributor 以上のロール）
- Azure CLI / azcopy / PowerShell
- Windows Server 2022 / 2019 の固定サイズ VHD ファイル（[入手方法](./docs/architecture/architecture-nested-hyperv.md#3-windows-server-vhd-の入手)）
- 必要に応じて GitHub Copilot ライセンス（モダナイズ系ステップで活用）

---

## はじめ方

1. [`docs/README.md`](./docs/README.md) で全体構成を確認
2. Step 0: 前提条件と VHD を準備
3. Step 1: 疑似オンプレ環境を構築し、Parts Unlimited をセットアップ
4. Step 2: クラウド基盤をデプロイ
5. Step 3: VPN & DNS を構成
6. Step 4〜10: Azure Arc / Azure Migrate / 各移行パターンを比較・クリーンアップ

---

## リポジトリ内の位置づけ

- `docs/` : ハンズオン ドキュメント
- `infra/` : ハンズオンで利用する Bicep / ARM / パラメータ / PowerShell 資産
  - `infra/nested/onprem/` : Nested Hyper-V 疑似オンプレ環境のテンプレート
  - `infra/nested/network/` : VPN & DNS テンプレート
  - `infra/cloud/` : クラウド基盤（Hub & Spoke）テンプレート

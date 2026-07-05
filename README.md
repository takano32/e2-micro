# e2-micro — GCP 無料枠 VM 運用メモ

GCP プロジェクト `takano32` の Always Free e2-micro インスタンスの構成・コスト・監視の記録。
作業履歴は [logs/](logs/) に日付ごとに残す。

## 現在の構成(2026-07-06 時点)

| 項目 | 値 |
|---|---|
| インスタンス | `e2-micro` @ **us-west1-a**(無料枠対象で日本に最も近いリージョン、日本から RTT 約105ms) |
| ディスク | pd-standard **30GB**(無料枠上限ちょうど。拡張すると課金) |
| 静的 IP | `e2-micro` = **8.231.236.96**(約 $3.6/月 — 唯一の固定課金) |
| 公開サービス | Caddy(80/443、`8-231-236-96.sslip.io`)→ cxx_chatwork(127.0.0.1:48080) |
| 常駐ワークロード | claude(Discord 連携)+ bun watcher(~/nilkani06)— **egress の主因** |

接続:

```
gcloud compute ssh --zone us-west1-a e2-micro --project takano32
```

## コスト構造

- **無料**: VM 本体・ディスク 30GB(us-west1 / us-central1 / us-east1 限定)
- **固定**: 外部 IPv4 $0.005/時 ≒ $3.6/月(2024-02 改定以降。無料枠 VM でも課金対象)
- **変動**: インターネット egress(Premium ティア)。無料は **1GB/月** のみ、超過分 約 $0.12/GB
  - 主因は claude / watcher の Anthropic API 送信(コンテキストのアップロード)
- **請求実績(2026年6月)**: **¥556**(定価 ¥901 − クレジット ¥345)
  - NIC 送信実測 30.7GB のうち課金 egress は約 21GB 相当(Google 宛て等の非課金分を除く)で、その大半がクレジットで相殺された
  - 結果として月額はほぼ静的 IP 代のみ。ただしクレジットの名目(無料枠/プロモーション/割引)は請求書明細で要確認 — プロモーションなら失効し得る
- 請求実額の確認: [請求レポート](https://console.cloud.google.com/billing/011D0B-92862C-4E6BD0/reports)(SKU 別表示で宛先大陸別の内訳が出る)
- インスタンス削除時は**静的 IP も削除すること**(未使用放置で倍額の $7.3/月)

## egress の内訳計測

### egress-report(宛先カテゴリ別・常設)

VM 上で:

```
egress-report
```

出力カテゴリ:

| カウンタ | 内容 | 課金 |
|---|---|---|
| `anthropic_api` | Anthropic API 直行(160.79.104.0/23) | 対象 |
| `cloudflare_discord` | Cloudflare 全レンジ(Discord ゲートウェイ、claude.ai 等) | 対象 |
| `ssh_replies` | SSH セッションへの応答 | 対象 |
| `internal_free` | メタデータ・VPC 内・リンクローカル | **対象外** |
| `other` | 上記以外(apt 更新など) | 対象 |

仕組み(設定ファイルのコピーは [vm/](vm/) に保存):

- `/etc/nftables.d/egress-acct.nft` — nftables カウンタ本体。**カウントのみで遮断はしない**。`nftables.service` 有効化済みで再起動後も自動ロード
- `/usr/local/bin/egress-report` — 読み出しコマンド(MB 表示)
- `/etc/cron.daily/egress-acct` — `/var/log/egress-acct.log` に日次スナップショット(JSON 1行/日)。カウンタは再起動で 0 に戻るが、履歴はここから復元する

### その他の監視手段

- `vnstat -m` — 月次の総量(NIC 全体、2026-07 導入以降)
- `sudo nethogs` — プロセス別のライブ送信量 / `sudo iftop` — 宛先別ライブ
- 過去分・VM 外から: Cloud Monitoring `instance/network/sent_bytes_count`(標準メトリクス・無料。Monitoring API 有効化済み)

## セキュリティ・安定運用

- SSH は鍵認証のみ(パスワード認証無効)+ **fail2ban**(sshd ジェイル)
- **zram** 有効(RAM 952MB の圧迫緩和)。自動セキュリティ更新 有効
- サービスアカウント: Compute デフォルト SA は VM が使用中のため**削除禁止**。App Engine デフォルト SA は未使用だが実害なしで残置

## 無料枠の要点(2026-07 確認)

- e2-micro 1台分の稼働時間 / pd-standard 30GB-月 / egress 1GB/月(北米発、中国・豪州宛を除く)
- 対象リージョンは us-west1 / us-central1 / us-east1 のみ(東京・大阪は対象外)
- 参照: [Free Tier ドキュメント](https://docs.cloud.google.com/free/docs/free-cloud-features) / [外部 IPv4 料金改定](https://cloud.google.com/vpc/pricing-announce-external-ips)

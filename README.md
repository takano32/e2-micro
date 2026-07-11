# e2-micro — GCP 無料枠 VM 運用メモ

GCP プロジェクト `takano32` の Always Free e2-micro インスタンスの構成・コスト・監視の記録。
作業履歴は [logs/](logs/) に日付ごとに残す。

## 現在の構成(2026-07-11 時点)

| 項目 | 値 |
|---|---|
| インスタンス | `e2-micro` @ **us-west1-a**(無料枠対象で日本に最も近いリージョン、日本から RTT 約110ms) |
| ディスク | pd-standard **30GB**(無料枠上限ちょうど。拡張すると課金) |
| 静的 IP | `e2-micro` = **35.212.198.233**(**STANDARD ティア**、2026-07-11 切替。旧 8.231.236.96 は解放済み) |
| 公開サービス | Caddy(80/443、`35-212-198-233.sslip.io`)→ cxx_chatwork(127.0.0.1:48080) |
| 常駐ワークロード | claude(Discord 連携)+ bun watcher(~/nilkani06)— **egress の主因** |

接続:

```
gcloud compute ssh --zone us-west1-a e2-micro --project takano32
```

## コスト構造(2026-07-11 の STANDARD ティア切替後)

- **無料**: VM 本体(月上限時間まで)・pd-standard 30GB — 無料枠で全額相殺を6月請求明細で確認済み
- **外部 IPv4**: 定価 $0.005/時だが、**現在はクレジットで全額相殺され実質 ¥0**(5月・6月とも Networking 行 ¥0)。クレジットの正式名称・恒常性は未確認なので、**毎月 Networking 行が ¥0 のままか確認する**
- **egress は STANDARD ティアに切替済み(2026-07-11)**: リージョンごとに**月 200GB まで無料**、超過 $0.085/GiB([料金](https://cloud.google.com/network-tiers/pricing))
  - 現ペース約 30〜50GiB/月 ≪ 200GB → **egress 課金も ¥0 の見込み**(Premium 時代は 6月 28.24GiB = ¥521、7月見込み ¥900 だった)
  - 日本からの RTT は約 110ms で Premium 時代(約105ms)とほぼ同等
  - vnstat / Cloud Monitoring で 200GB 超過に近づかないか監視。**8月請求で egress 行が ¥0 になったことを要確認**
- 6月請求の確定値(参考): ¥556 = egress 28.24GiB ¥521 + 移行スナップショット ¥34 + peering ¥1(SKU 明細で照合済み)
- 請求実額の確認: [請求レポート](https://console.cloud.google.com/billing/011D0B-92862C-4E6BD0/reports)(SKU 別表示で宛先大陸別の内訳が出る)
- インスタンス削除時は**静的 IP `e2-micro` も削除すること**(未使用放置で倍額の $7.3/月)

## egress の監視

- `vnstat -m` — 月次の総量(NIC 全体、2026-07 導入以降)。200GB/月の Standard 無料枠に対する現在値の確認はこれで十分
- `sudo nethogs` — プロセス別のライブ送信量 / `sudo iftop` — 宛先別ライブ
- 過去分・VM 外から: Cloud Monitoring `instance/network/sent_bytes_count`(標準メトリクス・無料。Monitoring API 有効化済み)

かつて設置していた nftables 宛先別カウンタ(`egress-report`)は、Standard 化で内訳把握の必要がなくなり **2026-07-11 に撤去済み**(最終集計: 課金対象送信の約 98% が Anthropic API 直行。詳細は [logs/2026-07-11.md](logs/2026-07-11.md))。

## セキュリティ・安定運用

- SSH は鍵認証のみ(パスワード認証無効)+ **fail2ban**(sshd ジェイル)
- **zram** 有効(RAM 952MB の圧迫緩和)。自動セキュリティ更新 有効
- サービスアカウント: Compute デフォルト SA は VM が使用中のため**削除禁止**。App Engine デフォルト SA は未使用だが実害なしで残置

## 無料枠の要点(2026-07 確認)

- e2-micro 1台分の稼働時間 / pd-standard 30GB-月 / egress 1GB/月(北米発、中国・豪州宛を除く)
- 対象リージョンは us-west1 / us-central1 / us-east1 のみ(東京・大阪は対象外)
- 参照: [Free Tier ドキュメント](https://docs.cloud.google.com/free/docs/free-cloud-features) / [外部 IPv4 料金改定](https://cloud.google.com/vpc/pricing-announce-external-ips)

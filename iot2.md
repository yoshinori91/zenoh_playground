## 全体設計 最終確定版

| 項目 | 決定事項 |
|------|----------|
| アカウント構成 | シングルアカウント |
| 環境分離 | Thing Group + ポリシー（dev/stg/prod） |
| CA | 擬似プライベートCA（Secret Manager） |
| 証明書発行 | Certificate Provider（Lambda） |
| 初回プロビジョニング | Fleet Provisioning + クレーム証明書 |
| クレーム証明書 | 初回完了後に削除 |
| デバイス証明書 | 1デバイス1証明書（環境をまたいで使い続ける） |
| 環境移行 | Thing Group・ポリシー切り替えのみ（証明書はそのまま） |
| 証明書ローテーション | Defender → SNS → Lambda → Certificate Provider |
| デバイス | Raspberry Pi / Docker compose |
| 秘密鍵 | デバイスで生成・外に出さない |
| 展開規模 | 1000台・海外・現地作業なし |

`/etc/iot/certs/` に証明書が置かれている場合、そのパスをそのまま各設定に反映するだけです。変更が必要な箇所を整理します。

---

## 1. Greengrass インストール時の設定

Greengrass Nucleus を Fleet Provisioning 済みの証明書で起動する場合、`config.yaml` を手動で作成してからインストーラを `--init-config` で実行します。

```yaml
# /tmp/greengrass-config.yaml
---
system:
  certificateFilePath: "/etc/iot/certs/device.pem"
  privateKeyPath: "/etc/iot/certs/private.key"
  rootCaPath: "/etc/iot/certs/AmazonCA.pem"
  thingName: "robot-XXXX"  # Fleet Provisioning で生成された Thing 名

services:
  aws.greengrass.Nucleus:
    configuration:
      awsRegion: "ap-northeast-1"
      iotCredEndpoint: "XXXX.credentials.iot.ap-northeast-1.amazonaws.com"
      iotDataEndpoint: "XXXX-ats.iot.ap-northeast-1.amazonaws.com"
      mqtt:
        port: 8883
```

```bash
sudo java -Droot="/greengrass/v2" \
  -Dlog.store=FILE \
  -jar ./GreengrassInstaller/lib/Greengrass.jar \
  --init-config /tmp/greengrass-config.yaml \
  --component-default-user ggc_user:ggc_group \
  --setup-system-service false
```

`--provision true` を使わず `--init-config` を使うのがポイントです。`--provision true` にすると Greengrass が新たに証明書を自動生成してしまい、Fleet Provisioning の証明書が上書きされます。

---

## 2. Greengrass の証明書ファイルへのアクセス権限

Greengrass は `ggc_user` で動くため、`/etc/iot/certs/` への読み取り権限が必要です。

```bash
sudo chown root:ggc_group /etc/iot/certs/
sudo chmod 750 /etc/iot/certs/

sudo chown root:ggc_group /etc/iot/certs/device.pem
sudo chown root:ggc_group /etc/iot/certs/private.key
sudo chown root:ggc_group /etc/iot/certs/AmazonCA.pem
sudo chmod 640 /etc/iot/certs/device.pem
sudo chmod 640 /etc/iot/certs/private.key
sudo chmod 640 /etc/iot/certs/AmazonCA.pem
```

---

## 3. KVS コンポーネントのデプロイ設定

証明書パスを `/etc/iot/certs/` に変えるだけです。

```json
{
  "targetArn": "arn:aws:iot:ap-northeast-1:ACCOUNT_ID:thing/robot-XXXX",
  "deploymentName": "kvs-webrtc-streams-deployment",
  "components": {
    "aws.greengrass.KinesisVideoStreamsWebRTC": {
      "componentVersion": "2.0.2",
      "configurationUpdate": {
        "merge": "{\"WebRTCConfig\":{\"ChannelName\":\"MyWSLCamera\",\"ChannelRoleAlias\":\"GreengrassKVSRoleAlias\",\"Region\":\"ap-northeast-1\",\"GStreamerPipelineForStream\":\"videotestsrc pattern=ball ! video/x-raw,width=640,height=480,framerate=15/1 ! videoconvert ! x264enc bframes=0 key-int-max=30 ! video/x-h264,stream-format=avc,alignment=au ! appsink sync=false name=appsink-video\"},\"Certificates\":{\"CAFilePath\":\"/etc/iot/certs/AmazonCA.pem\"}}"
      }
    },
    "aws.greengrass.KinesisVideoStreams": {
      "componentVersion": "7.1.0",
      "configurationUpdate": {
        "merge": "{\"StreamName\":\"MyWSLCameraStream\",\"RoleAlias\":\"GreengrassKVSRoleAlias\",\"Region\":\"ap-northeast-1\",\"GStreamerPipeline\":\"videotestsrc pattern=ball ! video/x-raw,width=640,height=480,framerate=15/1 ! videoconvert ! x264enc bframes=0 key-int-max=30 ! video/x-h264,stream-format=avc,alignment=au ! kvssink stream-name=MyWSLCameraStream aws-region=ap-northeast-1 iot-certificate=\\\"iot-certificate,endpoint=data.credentials.iot.ap-northeast-1.amazonaws.com,cert-path=/etc/iot/certs/device.pem,key-path=/etc/iot/certs/private.key,ca-path=/etc/iot/certs/AmazonCA.pem,role-aliases=GreengrassKVSRoleAlias\\\"\"}"
      }
    }
  }
}
```

---

## 4. Certificate Rotation Component のパス設定

証明書更新時に `/etc/iot/certs/` 以下を更新するよう明示します。

```json
"aws.greengrass.CertificateRotation": {
  "componentVersion": "1.0.0",
  "configurationUpdate": {
    "merge": "{\"CertificateFilePath\":\"/etc/iot/certs/device.pem\",\"PrivateKeyPath\":\"/etc/iot/certs/private.key\"}"
  }
}
```

ただし Certificate Rotation Component はデフォルトで Greengrass Nucleus が管理する証明書（`config.yaml` 記載のパス）を更新対象とします。`config.yaml` で `/etc/iot/certs/` を指定していれば自動的にそちらが更新されるため、上記の明示設定は念のための確認用です。

---

## 5. 証明書更新後の KVS 再起動

`kvssink` は起動時に証明書を読み込むため、更新後に KVS Streams コンポーネントの再起動が必要です。IoT Jobs と組み合わせて自動化する場合の流れはこうなります。

```
IoT Jobs 発行
  → Certificate Rotation Component が /etc/iot/certs/ の証明書を更新
  → Greengrass Nucleus 再接続
  → KVS Streams Component 再起動（Jobs のポストステップ or カスタムコンポーネントでトリガー）
```

再起動のトリガーはカスタムコンポーネントから IPC 経由で行えます。

```python
import awsiot.greengrasscoreipc as greengrasscoreipc
from awsiot.greengrasscoreipc.model import RestartComponentRequest

client = greengrasscoreipc.connect()
request = RestartComponentRequest(
    component_name="aws.greengrass.KinesisVideoStreams"
)
client.new_restart_component().activate(request).result(timeout=10)
```

---

## 変更箇所サマリー

```
1. Greengrass インストール → --init-config で既存証明書パスを指定（--provision true は使わない）
2. /etc/iot/certs/ の所有者・パーミッションを ggc_group に付与
3. KVS コンポーネントの cert-path / key-path / ca-path を /etc/iot/certs/ に変更
4. Certificate Rotation の更新対象パスが config.yaml と一致していることを確認
5. 証明書更新後に KVS Streams を再起動する仕組みを用意
```

既存の Fleet Provisioning の証明書配置をまったく変える必要がなく、Greengrass 側がそのパスを参照するように設定するだけなので、既存のデバイス管理フローへの影響は最小限です。

Fleet Provisioning のフローの中で Greengrass をインストールする場所は、**Fleet Provisioning が完了した直後、デバイス側のブートストラップスクリプト**の中です。

---

## フロー全体像

```
デバイス起動
  │
  ▼
Claim証明書でIoT Coreに接続
  │
  ▼
Fleet Provisioning Template 実行
  ├─ Thing 作成
  ├─ デバイス証明書発行 → /etc/iot/certs/ に保存
  ├─ IoT Policy アタッチ
  └─ Thing Group への追加
  │
  ▼
★ ここで Greengrass インストール ★
（デバイス側ブートストラップスクリプトの続き）
  │
  ▼
Greengrass 起動
```

Fleet Provisioning Template 自体はクラウド側のリソース作成だけを行うものなので、Greengrass のインストールはあくまで**デバイス側のスクリプト**で行います。

---

## デバイス側ブートストラップスクリプトの例

```bash
#!/bin/bash
# /usr/local/bin/bootstrap.sh
# デバイス初回起動時に1回だけ実行される想定

set -e

REGION="ap-northeast-1"
CERTS_DIR="/etc/iot/certs"
GREENGRASS_ROOT="/greengrass/v2"

# -----------------------------------------------
# Step 1: Fleet Provisioning 実行
# （Claim証明書は事前にデバイスに焼いておく）
# -----------------------------------------------
python3 /usr/local/bin/fleet_provisioning.py \
  --region $REGION \
  --claim-cert $CERTS_DIR/claim.pem \
  --claim-key $CERTS_DIR/claim.key \
  --ca $CERTS_DIR/AmazonCA.pem \
  --template-name MyFleetTemplate \
  --output-cert $CERTS_DIR/device.pem \
  --output-key $CERTS_DIR/private.key

# fleet_provisioning.py の完了時点で
# /etc/iot/certs/device.pem
# /etc/iot/certs/private.key
# /etc/iot/certs/AmazonCA.pem
# が揃っている状態

# -----------------------------------------------
# Step 2: Thing 名を取得
# （Fleet Provisioning の応答から保存しておく）
# -----------------------------------------------
THING_NAME=$(cat /var/lib/iot/thing_name)

# -----------------------------------------------
# Step 3: IoT エンドポイント取得
# -----------------------------------------------
IOT_DATA_ENDPOINT=$(aws iot describe-endpoint \
  --endpoint-type iot:Data-ATS \
  --region $REGION \
  --query endpointAddress --output text)

IOT_CRED_ENDPOINT=$(aws iot describe-endpoint \
  --endpoint-type iot:CredentialProvider \
  --region $REGION \
  --query endpointAddress --output text)

# -----------------------------------------------
# Step 4: Greengrass config.yaml を生成
# -----------------------------------------------
mkdir -p /tmp/greengrass-init

cat > /tmp/greengrass-init/config.yaml << EOF
---
system:
  certificateFilePath: "${CERTS_DIR}/device.pem"
  privateKeyPath: "${CERTS_DIR}/private.key"
  rootCaPath: "${CERTS_DIR}/AmazonCA.pem"
  thingName: "${THING_NAME}"

services:
  aws.greengrass.Nucleus:
    configuration:
      awsRegion: "${REGION}"
      iotCredEndpoint: "${IOT_CRED_ENDPOINT}"
      iotDataEndpoint: "${IOT_DATA_ENDPOINT}"
      mqtt:
        port: 8883
EOF

# -----------------------------------------------
# Step 5: Greengrass インストーラー取得 & インストール
# -----------------------------------------------
curl -s https://d2s8p88vqu9w66.cloudfront.net/releases/greengrass-nucleus-latest.zip \
  -o /tmp/greengrass-nucleus-latest.zip
unzip -q /tmp/greengrass-nucleus-latest.zip -d /tmp/GreengrassInstaller

sudo java -Droot="${GREENGRASS_ROOT}" \
  -Dlog.store=FILE \
  -jar /tmp/GreengrassInstaller/lib/Greengrass.jar \
  --init-config /tmp/greengrass-init/config.yaml \
  --component-default-user ggc_user:ggc_group \
  --setup-system-service true  # 実機は systemd サービスとして登録
                                # WSL2 の場合は false

# -----------------------------------------------
# Step 6: 証明書パーミッション設定
# -----------------------------------------------
chown root:ggc_group ${CERTS_DIR}/device.pem
chown root:ggc_group ${CERTS_DIR}/private.key
chown root:ggc_group ${CERTS_DIR}/AmazonCA.pem
chmod 640 ${CERTS_DIR}/device.pem
chmod 640 ${CERTS_DIR}/private.key
chmod 640 ${CERTS_DIR}/AmazonCA.pem

echo "Bootstrap complete: Greengrass installed as ${THING_NAME}"
```

---

## WSL2 検証環境での実行順序

実機（Raspberry Pi 等）と異なり WSL2 では systemd が使えないため、起動順序を手動で管理します。

```bash
# 1. Fleet Provisioning スクリプトを実行（証明書取得）
python3 /usr/local/bin/fleet_provisioning.py ...

# 2. Greengrass インストール（--setup-system-service false）
sudo java -Droot="/greengrass/v2" \
  -jar /tmp/GreengrassInstaller/lib/Greengrass.jar \
  --init-config /tmp/greengrass-init/config.yaml \
  --component-default-user ggc_user:ggc_group \
  --setup-system-service false

# 3. Greengrass 手動起動
sudo /greengrass/v2/alts/current/distro/bin/loader &

# 4. 起動確認
tail -f /greengrass/v2/logs/greengrass.log
```

---

## 重要ポイントのまとめ

| ポイント | 理由 |
|---|---|
| Fleet Provisioning 完了後に Greengrass をインストールする | デバイス証明書が `/etc/iot/certs/` に揃ってから `--init-config` を渡すため |
| `--provision true` は使わない | 使うと Greengrass が新たに証明書を生成し Fleet Provisioning の証明書と競合する |
| `--init-config` に証明書パスを明示する | Greengrass が `/etc/iot/certs/` を参照するよう紐付けるため |
| Thing 名は Fleet Provisioning の応答から取得して渡す | `config.yaml` の `thingName` が IoT Core の Thing と一致している必要があるため |

Fleet Provisioning → Greengrass インストールの順序さえ守れば、証明書を二重管理せずにそのまま使い回せます。

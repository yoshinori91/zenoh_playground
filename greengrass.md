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

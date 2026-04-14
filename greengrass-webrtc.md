WSL上でGreengrassを動かしてKVS WebRTC + KVS Streams両方を検証する構成の手順を説明します。

## 全体の流れ

1. WSL2 環境準備
2. Greengrass v2 インストール
3. IAM / IoT Core リソース作成
4. KVS WebRTC Component デプロイ
5. KVS Streams Component デプロイ
6. カメラ映像の分岐（GStreamer tee）

---

## 1. WSL2 環境準備

```bash
# Ubuntu 22.04 推奨
wsl --install -d Ubuntu-22.04

# 必要パッケージ
sudo apt update && sudo apt install -y \
  openjdk-11-jdk \
  python3 python3-pip \
  curl unzip git \
  gstreamer1.0-tools \
  gstreamer1.0-plugins-base \
  gstreamer1.0-plugins-good \
  gstreamer1.0-plugins-bad \
  gstreamer1.0-plugins-ugly \
  gstreamer1.0-libav \
  libgstreamer1.0-dev \
  libgstreamer-plugins-base1.0-dev
```

WSL2 はカメラデバイスを直接認識しないため、映像ソースは `videotestsrc`（テストパターン）か、Windows側の映像を UDP 経由で受け取る形にします。

```bash
# テスト用: videotestsrc でパイプライン確認
gst-launch-1.0 videotestsrc ! videoconvert ! autovideosink
```

---

## 2. Greengrass v2 インストール

```bash
# インストーラー取得
curl -s https://d2s8p88vqu9w66.cloudfront.net/releases/greengrass-nucleus-latest.zip \
  -o greengrass-nucleus-latest.zip
unzip greengrass-nucleus-latest.zip -d GreengrassInstaller

# インストール（後述のIAM設定後に実行）
sudo -E java -Droot="/greengrass/v2" \
  -Dlog.store=FILE \
  -jar ./GreengrassInstaller/lib/Greengrass.jar \
  --aws-region ap-northeast-1 \
  --thing-name MyWSLGreengrassCore \
  --thing-group-name MyGreengrassGroup \
  --component-default-user ggc_user:ggc_group \
  --provision true \
  --setup-system-service false \
  --deploy-dev-tools true
```

`--setup-system-service false` にするのは WSL2 では systemd が制限されるためで、手動起動で問題ありません。

---

## 3. IAM / IoT Core リソース作成

AWS CLI で必要なリソースをまとめて作ります。

```bash
REGION=ap-northeast-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
THING_NAME=MyWSLGreengrassCore

# --- IoT Thing / 証明書はGreengrass自動プロビジョニングで作成済み ---

# KVS用 IAM ロール
aws iam create-role \
  --role-name GreengrassKVSRole \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Principal":{"Service":"credentials.iot.amazonaws.com"},
      "Action":"sts:AssumeRole"
    }]
  }'

# KVS WebRTC + Streams 両方の権限を付与
aws iam put-role-policy \
  --role-name GreengrassKVSRole \
  --policy-name KVSFullAccess \
  --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Action":[
        "kinesisvideo:ConnectAsMaster",
        "kinesisvideo:GetSignalingChannelEndpoint",
        "kinesisvideo:CreateSignalingChannel",
        "kinesisvideo:DescribeSignalingChannel",
        "kinesisvideo:GetIceServerConfig",
        "kinesisvideo:PutMedia",
        "kinesisvideo:CreateStream",
        "kinesisvideo:DescribeStream",
        "kinesisvideo:GetDataEndpoint"
      ],
      "Resource":"*"
    }]
  }'

# IoT Role Alias 作成
aws iot create-role-alias \
  --role-alias GreengrassKVSRoleAlias \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/GreengrassKVSRole \
  --region $REGION

# IoT Policy に Role Alias の assumeRoleWithCertificate を追加
THING_CERT_ARN=$(aws iot list-thing-principals \
  --thing-name $THING_NAME \
  --query principals[0] --output text)

aws iot create-policy \
  --policy-name GreengrassKVSIoTPolicy \
  --policy-document "{
    \"Version\":\"2012-10-17\",
    \"Statement\":[{
      \"Effect\":\"Allow\",
      \"Action\":\"iot:AssumeRoleWithCertificate\",
      \"Resource\":\"arn:aws:iot:${REGION}:${ACCOUNT_ID}:rolealias/GreengrassKVSRoleAlias\"
    }]
  }"

aws iot attach-policy \
  --policy-name GreengrassKVSIoTPolicy \
  --target $THING_CERT_ARN
```

---

## 4. KVS Signaling Channel と Stream の作成

```bash
# WebRTC用 Signaling Channel
aws kinesisvideo create-signaling-channel \
  --channel-name MyWSLCamera \
  --channel-type SINGLE_MASTER \
  --region $REGION

# Streams用 Video Stream
aws kinesisvideo create-stream \
  --stream-name MyWSLCameraStream \
  --data-retention-in-hours 168 \
  --region $REGION
```

---

## 5. Greengrass コンポーネントのデプロイ設定

`deployment.json` を作成してデプロイします。

```json
{
  "targetArn": "arn:aws:iot:ap-northeast-1:ACCOUNT_ID:thinggroup/MyGreengrassGroup",
  "deploymentName": "kvs-webrtc-streams-deployment",
  "components": {
    "aws.greengrass.KinesisVideoStreamsWebRTC": {
      "componentVersion": "2.0.2",
      "configurationUpdate": {
        "merge": "{\"Certificates\":{\"CAFilePath\":\"/greengrass/v2/rootCA.pem\"},\"StreamManager\":{\"DataEndpointType\":\"HTTPS\"},\"WebRTCConfig\":{\"ChannelName\":\"MyWSLCamera\",\"ChannelRoleAlias\":\"GreengrassKVSRoleAlias\",\"Region\":\"ap-northeast-1\",\"GStreamerPipelineForStream\":\"videotestsrc pattern=ball ! video/x-raw,width=640,height=480,framerate=15/1 ! videoconvert ! x264enc bframes=0 key-int-max=30 ! video/x-h264,stream-format=avc,alignment=au ! appsink sync=false name=appsink-video\"}}"
      }
    },
    "aws.greengrass.KinesisVideoStreams": {
      "componentVersion": "7.1.0",
      "configurationUpdate": {
        "merge": "{\"StreamName\":\"MyWSLCameraStream\",\"RoleAlias\":\"GreengrassKVSRoleAlias\",\"Region\":\"ap-northeast-1\",\"GStreamerPipeline\":\"videotestsrc pattern=ball ! video/x-raw,width=640,height=480,framerate=15/1 ! videoconvert ! x264enc bframes=0 key-int-max=30 ! video/x-h264,stream-format=avc,alignment=au ! kvssink stream-name=MyWSLCameraStream aws-region=ap-northeast-1 iot-certificate=\\\"iot-certificate,endpoint=data.credentials.iot.ap-northeast-1.amazonaws.com,cert-path=/greengrass/v2/thingCert.crt,key-path=/greengrass/v2/privKey.key,ca-path=/greengrass/v2/rootCA.pem,role-aliases=GreengrassKVSRoleAlias\\\"\"}"
      }
    }
  }
}
```

```bash
aws greengrassv2 create-deployment \
  --cli-input-json file://deployment.json \
  --region $REGION
```

---

## 6. 映像を1ソースから両方に分岐させる場合

両コンポーネントのパイプラインを個別に `videotestsrc` から立ち上げるのが最もシンプルですが、実カメラを1台使いたい場合は GStreamer の `tee` で分岐させたカスタムコンポーネントを作る方法が確実です。

```bash
# tee で WebRTC と kvssink に同時送出するパイプライン例（動作確認用）
gst-launch-1.0 \
  videotestsrc pattern=ball ! \
  video/x-raw,width=640,height=480,framerate=15/1 ! \
  videoconvert ! \
  tee name=t \
    t. ! queue ! x264enc bframes=0 key-int-max=30 ! \
       video/x-h264,stream-format=avc,alignment=au ! \
       kvssink stream-name=MyWSLCameraStream \
         aws-region=ap-northeast-1 \
         iot-certificate="iot-certificate,endpoint=data.credentials.iot.ap-northeast-1.amazonaws.com,cert-path=/greengrass/v2/thingCert.crt,key-path=/greengrass/v2/privKey.key,ca-path=/greengrass/v2/rootCA.pem,role-aliases=GreengrassKVSRoleAlias" \
    t. ! queue ! x264enc bframes=0 key-int-max=30 ! \
       video/x-h264,stream-format=avc,alignment=au ! \
       fakesink
       # WebRTC側は appsink 経由でSDK内部が処理するためここはfakesink
```

---

## 7. 動作確認

```bash
# Greengrass 起動
sudo /greengrass/v2/alts/current/distro/bin/loader

# ログ確認
tail -f /greengrass/v2/logs/aws.greengrass.KinesisVideoStreamsWebRTC.log
tail -f /greengrass/v2/logs/aws.greengrass.KinesisVideoStreams.log

# KVS Streams への取り込み確認
aws kinesisvideo describe-stream \
  --stream-name MyWSLCameraStream \
  --region $REGION

# WebRTC 接続テスト（ブラウザから）
# https://awslabs.github.io/amazon-kinesis-video-streams-webrtc-sdk-js/examples/index.html
# Channel name: MyWSLCamera, Region: ap-northeast-1 を入力してViewer接続
```

---

## WSL2 特有の注意点

| 項目 | 注意 |
|---|---|
| カメラデバイス | WSL2 は `/dev/video0` を直接使えないため、検証は `videotestsrc` 推奨。実カメラを使いたい場合は `usbipd-win` でUSBパススルーが必要 |
| systemd | WSL2 のデフォルトでは systemd が無効なため Greengrass をサービス登録せず手動起動 |
| ネットワーク | NAT 構成のため TURN 経由になるケースが多い。WebRTC の ICE は問題なく通る |
| Java | Greengrass Nucleus は JDK 11 必須。`java -version` で確認してから実行 |

---

まずは `videotestsrc` でパイプラインと認証が通ることを確認してから、実カメラ（`usbipd-win`）や動画ファイル（`filesrc`）への切り替えを行うのがスムーズです。詰まりそうなポイントがあればどこでも聞いてください。

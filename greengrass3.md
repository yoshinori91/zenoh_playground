であれば、最初から Greengrass ありで検証する方針は正しいと思います。

検証フェーズで Greengrass を使っておくメリットは明確で、本番移行時に構成を作り直す必要がなく、コンポーネントの設定調整だけで済みます。

---

## 検証の進め方として推奨する順序

```
Step 1: Greengrass 単体の動作確認
  └─ WSL2 または Jetson に Greengrass をインストール
  └─ IoT Core への MQTT 接続が通ることを確認

Step 2: Fleet Provisioning との接続確認
  └─ /etc/iot/certs/ の証明書で Greengrass が起動できることを確認
  └─ config.yaml のパスが正しく認識されていることを確認

Step 3: KVS WebRTC 単体の動作確認
  └─ videotestsrc でシグナリングチャネルへの接続を確認
  └─ ブラウザの Viewer SDK から映像が見えることを確認

Step 4: KVS Streams 単体の動作確認
  └─ videotestsrc で PutMedia が通ることを確認
  └─ AWS コンソールで HLS 再生できることを確認

Step 5: 両方の同時動作確認
  └─ tee パイプラインで WebRTC + Streams 同時送信
  └─ リソース使用量（CPU / メモリ）を確認

Step 6: カスタムコンポーネント（KVSSessionManager）の動作確認
  └─ 55分後に WebRTC セッションが自動更新されることを確認
  └─ IoT Jobs 経由の証明書更新フローを確認
```

Step を細かく分けているのは、問題が起きたときにどのレイヤーで詰まっているかをすぐ特定できるようにするためです。いきなり全部まとめて動かすとデバッグが大変になります。

---

では Step 1 から進めます。

まず Jetson と WSL2、どちらで進めますか？

それによって以下が変わります。

| 項目 | Jetson | WSL2 |
|---|---|---|
| systemd | 使用可能 | デフォルト無効 |
| Java インストール | apt で可能 | apt で可能 |
| Greengrass サービス登録 | `--setup-system-service true` | `--setup-system-service false` |
| 証明書パス | `/etc/iot/certs/` をそのまま使用 | 同左 |
| Fleet Provisioning 実行 | 既存スクリプトをそのまま使用 | 同左 |

WSL2 の方が手軽に試せますが、Jetson の方が本番に近い環境です。どちらでしょうか？

では Greengrass のインストールに進みます。

---

## Step 1-1: ggc_user / ggc_group の作成

```bash
sudo adduser --system --no-create-home ggc_user
sudo addgroup --system ggc_group
sudo adduser ggc_user ggc_group
```

---

## Step 1-2: Greengrass インストーラーの取得

```bash
cd /tmp
curl -s https://d2s8p88vqu9w66.cloudfront.net/releases/greengrass-nucleus-latest.zip \
    -o greengrass-nucleus-latest.zip
unzip -q greengrass-nucleus-latest.zip -d GreengrassInstaller

# バージョン確認
java -jar /tmp/GreengrassInstaller/lib/Greengrass.jar --version
```

---

## Step 1-3: config.yaml の作成

Fleet Provisioning 済みの証明書をそのまま使います。

```bash
# IoT エンドポイントを取得
IOT_DATA_ENDPOINT=$(aws iot describe-endpoint \
    --endpoint-type iot:Data-ATS \
    --region ap-northeast-1 \
    --query endpointAddress --output text)

IOT_CRED_ENDPOINT=$(aws iot describe-endpoint \
    --endpoint-type iot:CredentialProvider \
    --region ap-northeast-1 \
    --query endpointAddress --output text)

echo "Data: $IOT_DATA_ENDPOINT"
echo "Cred: $IOT_CRED_ENDPOINT"

# Thing 名を確認（Fleet Provisioning で生成されたもの）
THING_NAME=$(cat /var/lib/iot/thing_name)
echo "Thing: $THING_NAME"
```

エンドポイントと Thing 名が取得できたら config.yaml を生成します。

```bash
sudo mkdir -p /tmp/greengrass-init

sudo tee /tmp/greengrass-init/config.yaml << EOF
---
system:
  certificateFilePath: "/etc/iot/certs/device.pem"
  privateKeyPath: "/etc/iot/certs/private.key"
  rootCaPath: "/etc/iot/certs/AmazonCA.pem"
  thingName: "${THING_NAME}"

services:
  aws.greengrass.Nucleus:
    configuration:
      awsRegion: "ap-northeast-1"
      iotCredEndpoint: "${IOT_CRED_ENDPOINT}"
      iotDataEndpoint: "${IOT_DATA_ENDPOINT}"
      mqtt:
        port: 8883
EOF

# 内容確認
cat /tmp/greengrass-init/config.yaml
```

---

## Step 1-4: Greengrass インストール

```bash
sudo java -Droot="/greengrass/v2" \
    -Dlog.store=FILE \
    -jar /tmp/GreengrassInstaller/lib/Greengrass.jar \
    --init-config /tmp/greengrass-init/config.yaml \
    --component-default-user ggc_user:ggc_group \
    --setup-system-service false
```

---

## Step 1-5: 証明書パーミッション設定

```bash
sudo chown root:ggc_group /etc/iot/certs/device.pem
sudo chown root:ggc_group /etc/iot/certs/private.key
sudo chown root:ggc_group /etc/iot/certs/AmazonCA.pem
sudo chmod 640 /etc/iot/certs/device.pem
sudo chmod 640 /etc/iot/certs/private.key
sudo chmod 640 /etc/iot/certs/AmazonCA.pem
```

---

## Step 1-6: Greengrass 起動

```bash
sudo /greengrass/v2/alts/current/distro/bin/loader &

# ログを確認
tail -f /greengrass/v2/logs/greengrass.log
```

以下のようなログが出れば IoT Core への接続成功です。

```
...Nucleus started with AWS IoT Core connectivity...
```

---

上から順に実行してみてください。`config.yaml` の生成あたりで `THING_NAME` が取得できない場合は、Thing 名をどこかに保存していますか？

Step 1 が正常に完了したということですね。

では Step 2「Fleet Provisioning との接続確認」に進みます。

---

## Step 2-1: Greengrass が正しい証明書を使っているか確認

```bash
# Greengrass が読み込んでいる config.yaml を確認
sudo cat /greengrass/v2/config/config.yaml
```

以下の3つのパスが `/etc/iot/certs/` を指していることを確認します。

```yaml
system:
  certificateFilePath: "/etc/iot/certs/device.pem"   # ここ
  privateKeyPath: "/etc/iot/certs/private.key"        # ここ
  rootCaPath: "/etc/iot/certs/AmazonCA.pem"           # ここ
  thingName: "robot-XXXX"
```

---

## Step 2-2: IoT Core への MQTT 接続確認

```bash
# Greengrass のログで接続状態を確認
sudo grep -E "Connected|MQTT|certificate" \
    /greengrass/v2/logs/greengrass.log | tail -20
```

以下のようなログが出ていれば接続成功です。

```
...Successfully connected to AWS IoT Core...
...MQTT connection established...
```

---

## Step 2-3: AWS コンソール側での確認

IoT Core のコンソールから Thing が接続中になっているか確認します。

```bash
# CLI で確認する場合
aws iot describe-thing \
    --thing-name $(sudo grep thingName /greengrass/v2/config/config.yaml \
        | awk '{print $2}' | tr -d '"') \
    --region ap-northeast-1
```

---

## Step 2-4: Greengrass の Thing が Thing Group に所属しているか確認

```bash
THING_NAME=$(sudo grep thingName /greengrass/v2/config/config.yaml \
    | awk '{print $2}' | tr -d '"')

aws iot list-thing-groups-for-thing \
    --thing-name $THING_NAME \
    --region ap-northeast-1
```

Thing Group に所属していることを確認します。後の Step でデプロイ先として使います。

---

## Step 2-5: Greengrass コンポーネントの動作確認

```bash
# デプロイ済みコンポーネント一覧
sudo /greengrass/v2/bin/greengrass-cli \
    component list
```

`aws.greengrass.Nucleus` が `RUNNING` になっていれば OK です。

```
Component Name: aws.greengrass.Nucleus
    Version: 2.X.X
    State: RUNNING
```

---

各コマンドの結果はどうでしたか？

では Step 3「KVS WebRTC 単体の動作確認」に進みます。

---

## Step 3-1: 事前リソース作成

```bash
REGION=ap-northeast-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Signaling Channel 作成
aws kinesisvideo create-signaling-channel \
    --channel-name MyWSLCamera \
    --channel-type SINGLE_MASTER \
    --region $REGION

# 作成確認
aws kinesisvideo describe-signaling-channel \
    --channel-name MyWSLCamera \
    --region $REGION
```

---

## Step 3-2: IAM ロール / Role Alias 作成

```bash
# KVS 用 IAM ロール作成
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

# KVS WebRTC + Streams 両方の権限付与
aws iam put-role-policy \
    --role-name GreengrassKVSRole \
    --policy-name KVSPolicy \
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

# Role Alias 作成
aws iot create-role-alias \
    --role-alias GreengrassKVSRoleAlias \
    --role-arn arn:aws:iam::${ACCOUNT_ID}:role/GreengrassKVSRole \
    --region $REGION
```

---

## Step 3-3: IoT Policy に Role Alias 権限を追加

```bash
# Fleet Provisioning が適用したポリシー名を確認
THING_NAME=$(sudo grep thingName /greengrass/v2/config/config.yaml \
    | awk '{print $2}' | tr -d '"')

CERT_ARN=$(aws iot list-thing-principals \
    --thing-name $THING_NAME \
    --region $REGION \
    --query principals[0] --output text)

POLICY_NAME=$(aws iot list-attached-policies \
    --target $CERT_ARN \
    --region $REGION \
    --query policies[0].policyName --output text)

echo "Policy: $POLICY_NAME"

# 既存ポリシーの内容を確認
aws iot get-policy \
    --policy-name $POLICY_NAME \
    --region $REGION
```

既存のポリシー内容を確認してから `iot:AssumeRoleWithCertificate` を追加します。

```bash
# 新バージョンとして追加
aws iot create-policy-version \
    --policy-name $POLICY_NAME \
    --set-as-default \
    --policy-document "{
        \"Version\": \"2012-10-17\",
        \"Statement\": [
            {
                \"Effect\": \"Allow\",
                \"Action\": [
                    \"iot:Connect\",
                    \"iot:Publish\",
                    \"iot:Subscribe\",
                    \"iot:Receive\",
                    \"greengrass:*\"
                ],
                \"Resource\": \"*\"
            },
            {
                \"Effect\": \"Allow\",
                \"Action\": \"iot:AssumeRoleWithCertificate\",
                \"Resource\": \"arn:aws:iot:${REGION}:${ACCOUNT_ID}:rolealias/GreengrassKVSRoleAlias\"
            }
        ]
    }"
```

既存ポリシーのステートメントは Fleet Provisioning の内容に合わせて保持してください。

---

## Step 3-4: KVS WebRTC Component のデプロイ

```bash
THING_NAME=$(sudo grep thingName /greengrass/v2/config/config.yaml \
    | awk '{print $2}' | tr -d '"')

# デプロイ用 JSON 作成
cat > /tmp/kvs-webrtc-deployment.json << EOF
{
    "targetArn": "arn:aws:iot:${REGION}:${ACCOUNT_ID}:thing/${THING_NAME}",
    "deploymentName": "kvs-webrtc-deployment",
    "components": {
        "aws.greengrass.KinesisVideoStreamsWebRTC": {
            "componentVersion": "2.0.2",
            "configurationUpdate": {
                "merge": "{\"WebRTCConfig\":{\"ChannelName\":\"MyWSLCamera\",\"ChannelRoleAlias\":\"GreengrassKVSRoleAlias\",\"Region\":\"${REGION}\",\"GStreamerPipelineForStream\":\"videotestsrc pattern=ball ! video/x-raw,width=640,height=480,framerate=15/1 ! videoconvert ! x264enc bframes=0 key-int-max=30 ! video/x-h264,stream-format=avc,alignment=au ! appsink sync=false name=appsink-video\"},\"Certificates\":{\"CAFilePath\":\"/etc/iot/certs/AmazonCA.pem\"}}"
            }
        }
    }
}
EOF

# デプロイ実行
aws greengrassv2 create-deployment \
    --cli-input-json file:///tmp/kvs-webrtc-deployment.json \
    --region $REGION
```

---

## Step 3-5: デプロイ状況確認

```bash
# デプロイステータス確認
aws greengrassv2 list-deployments \
    --region $REGION \
    --query 'deployments[0].deploymentStatus'

# コンポーネントの状態確認
sudo /greengrass/v2/bin/greengrass-cli component list

# ログ確認
tail -f /greengrass/v2/logs/aws.greengrass.KinesisVideoStreamsWebRTC.log
```

`RUNNING` になれば OK です。

---

## Step 3-6: ブラウザから Viewer 接続テスト

以下の URL にアクセスします。

```
https://awslabs.github.io/amazon-kinesis-video-streams-webrtc-sdk-js/examples/index.html
```

入力項目はこのように設定します。

| 項目 | 値 |
|---|---|
| Region | ap-northeast-1 |
| Access Key ID | AWS アクセスキー |
| Secret Access Key | AWS シークレットキー |
| Channel Name | MyWSLCamera |
| Client ID | 任意の文字列 |
| Send Video | ON |
| Send Audio | OFF |

`Start Viewer` をクリックして videotestsrc のボールが動くテストパターンが見えれば Step 3 完了です。

---

Step 3-1 のリソース作成から順に実行してみてください。詰まった箇所があればログを貼っていただければ確認します。

では Step 4「KVS Streams 単体の動作確認」に進みます。

---

## Step 4-1: KVS Stream の作成

```bash
REGION=ap-northeast-1

# Stream 作成
aws kinesisvideo create-stream \
    --stream-name MyWSLCameraStream \
    --data-retention-in-hours 24 \
    --region $REGION

# 作成確認
aws kinesisvideo describe-stream \
    --stream-name MyWSLCameraStream \
    --region $REGION
```

---

## Step 4-2: kvssink GStreamer プラグインのインストール

WSL2 は x86_64 なのでソースビルドが必要です。

```bash
# ビルド依存パッケージ
sudo apt-get install -y \
    cmake \
    libssl-dev \
    libcurl4-openssl-dev \
    liblog4cplus-dev \
    libgstreamer1.0-dev \
    libgstreamer-plugins-base1.0-dev \
    gstreamer1.0-plugins-base \
    gstreamer1.0-plugins-good \
    gstreamer1.0-plugins-bad \
    gstreamer1.0-plugins-ugly \
    gstreamer1.0-libav \
    gstreamer1.0-tools

# KVS Producer SDK ソース取得
cd /tmp
git clone https://github.com/awslabs/amazon-kinesis-video-streams-producer-sdk-cpp.git
cd amazon-kinesis-video-streams-producer-sdk-cpp

# ビルド
mkdir build && cd build
cmake .. \
    -DBUILD_GSTREAMER_PLUGIN=ON \
    -DBUILD_JNI=FALSE
make -j$(nproc)

# プラグインをインストール
sudo cp libgstkvssink.so /usr/local/lib/gstreamer-1.0/

# GST_PLUGIN_PATH を永続化
echo 'export GST_PLUGIN_PATH=/usr/local/lib/gstreamer-1.0' \
    >> ~/.bashrc
source ~/.bashrc

# プラグイン確認
gst-inspect-1.0 kvssink
```

`kvssink` の情報が表示されれば OK です。

---

## Step 4-3: IoT Credential エンドポイントの確認

```bash
IOT_CRED_ENDPOINT=$(aws iot describe-endpoint \
    --endpoint-type iot:CredentialProvider \
    --region $REGION \
    --query endpointAddress --output text)

echo $IOT_CRED_ENDPOINT
```

---

## Step 4-4: kvssink で直接 PutMedia テスト

Greengrass を介さずにまず直接 GStreamer パイプラインで動作確認します。

```bash
export GST_PLUGIN_PATH=/usr/local/lib/gstreamer-1.0
export AWS_DEFAULT_REGION=ap-northeast-1

gst-launch-1.0 \
    videotestsrc pattern=ball duration=30000000000 ! \
    video/x-raw,width=640,height=480,framerate=15/1 ! \
    videoconvert ! \
    x264enc bframes=0 key-int-max=30 ! \
    video/x-h264,stream-format=avc,alignment=au ! \
    kvssink \
        stream-name=MyWSLCameraStream \
        aws-region=ap-northeast-1 \
        iot-certificate="iot-certificate,\
endpoint=${IOT_CRED_ENDPOINT},\
cert-path=/etc/iot/certs/device.pem,\
key-path=/etc/iot/certs/private.key,\
ca-path=/etc/iot/certs/AmazonCA.pem,\
role-aliases=GreengrassKVSRoleAlias"
```

30秒間 videotestsrc の映像を KVS Streams に送信します。

---

## Step 4-5: KVS Streams Component のデプロイ

直接パイプラインで動作確認できたら Greengrass コンポーネントとしてデプロイします。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
THING_NAME=$(sudo grep thingName /greengrass/v2/config/config.yaml \
    | awk '{print $2}' | tr -d '"')

cat > /tmp/kvs-streams-deployment.json << EOF
{
    "targetArn": "arn:aws:iot:${REGION}:${ACCOUNT_ID}:thing/${THING_NAME}",
    "deploymentName": "kvs-streams-deployment",
    "components": {
        "aws.greengrass.KinesisVideoStreams": {
            "componentVersion": "7.1.0",
            "configurationUpdate": {
                "merge": "{\"StreamName\":\"MyWSLCameraStream\",\"RoleAlias\":\"GreengrassKVSRoleAlias\",\"Region\":\"${REGION}\",\"GStreamerPipeline\":\"videotestsrc pattern=ball ! video/x-raw,width=640,height=480,framerate=15/1 ! videoconvert ! x264enc bframes=0 key-int-max=30 ! video/x-h264,stream-format=avc,alignment=au ! kvssink stream-name=MyWSLCameraStream aws-region=${REGION} iot-certificate=\\\"iot-certificate,endpoint=${IOT_CRED_ENDPOINT},cert-path=/etc/iot/certs/device.pem,key-path=/etc/iot/certs/private.key,ca-path=/etc/iot/certs/AmazonCA.pem,role-aliases=GreengrassKVSRoleAlias\\\"\"}"
            }
        }
    }
}
EOF

aws greengrassv2 create-deployment \
    --cli-input-json file:///tmp/kvs-streams-deployment.json \
    --region $REGION
```

---

## Step 4-6: デプロイ状況とログ確認

```bash
# コンポーネント状態確認
sudo /greengrass/v2/bin/greengrass-cli component list

# ログ確認
tail -f /greengrass/v2/logs/aws.greengrass.KinesisVideoStreams.log
```

---

## Step 4-7: AWS コンソールで HLS 再生確認

```bash
# HLS ストリーミング URL 取得
DATA_ENDPOINT=$(aws kinesisvideo get-data-endpoint \
    --stream-name MyWSLCameraStream \
    --api-name GET_HLS_STREAMING_SESSION_URL \
    --region $REGION \
    --query DataEndpoint --output text)

aws kinesis-video-archived-media get-hls-streaming-session-url \
    --stream-name MyWSLCameraStream \
    --playback-mode LIVE \
    --endpoint-url $DATA_ENDPOINT \
    --region $REGION \
    --query HLSStreamingSessionURL --output text
```

取得した URL を以下のいずれかで再生します。

| 方法 | 手順 |
|---|---|
| AWS コンソール | IoT Core → Kinesis Video Streams → ストリーム選択 → メディア再生 |
| VLC | `メディア → ネットワークストリームを開く` に URL を貼り付け |
| ブラウザ | hls.js デモページ `https://hls-js.netlify.app/demo/` に URL を貼り付け |

videotestsrc のボールが動くテストパターンが再生されれば Step 4 完了です。

---

Step 4-2 の kvssink ビルドが一番時間がかかります（10〜20分程度）。ビルド中にエラーが出た場合はログを貼っていただければ確認します。

では Step 5「WebRTC + KVS Streams 同時動作確認」に進みます。

---

## Step 5-1: 既存デプロイメントの確認

Step 3 と Step 4 で別々にデプロイしたコンポーネントを1つのデプロイメントに統合します。

```bash
# 現在のデプロイメント確認
aws greengrassv2 list-deployments \
    --region ap-northeast-1 \
    --query 'deployments[*].{name:deploymentName,status:deploymentStatus}'
```

---

## Step 5-2: 統合デプロイメントの作成

```bash
REGION=ap-northeast-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
THING_NAME=$(sudo grep thingName /greengrass/v2/config/config.yaml \
    | awk '{print $2}' | tr -d '"')
IOT_CRED_ENDPOINT=$(aws iot describe-endpoint \
    --endpoint-type iot:CredentialProvider \
    --region $REGION \
    --query endpointAddress --output text)

cat > /tmp/kvs-combined-deployment.json << EOF
{
    "targetArn": "arn:aws:iot:${REGION}:${ACCOUNT_ID}:thing/${THING_NAME}",
    "deploymentName": "kvs-combined-deployment",
    "components": {
        "aws.greengrass.KinesisVideoStreamsWebRTC": {
            "componentVersion": "2.0.2",
            "configurationUpdate": {
                "merge": "{\"WebRTCConfig\":{\"ChannelName\":\"MyWSLCamera\",\"ChannelRoleAlias\":\"GreengrassKVSRoleAlias\",\"Region\":\"${REGION}\",\"GStreamerPipelineForStream\":\"videotestsrc pattern=ball ! video/x-raw,width=640,height=480,framerate=15/1 ! videoconvert ! x264enc bframes=0 key-int-max=30 ! video/x-h264,stream-format=avc,alignment=au ! appsink sync=false name=appsink-video\"},\"Certificates\":{\"CAFilePath\":\"/etc/iot/certs/AmazonCA.pem\"}}"
            }
        },
        "aws.greengrass.KinesisVideoStreams": {
            "componentVersion": "7.1.0",
            "configurationUpdate": {
                "merge": "{\"StreamName\":\"MyWSLCameraStream\",\"RoleAlias\":\"GreengrassKVSRoleAlias\",\"Region\":\"${REGION}\",\"GStreamerPipeline\":\"videotestsrc pattern=ball ! video/x-raw,width=640,height=480,framerate=15/1 ! videoconvert ! x264enc bframes=0 key-int-max=30 ! video/x-h264,stream-format=avc,alignment=au ! kvssink stream-name=MyWSLCameraStream aws-region=${REGION} iot-certificate=\\\"iot-certificate,endpoint=${IOT_CRED_ENDPOINT},cert-path=/etc/iot/certs/device.pem,key-path=/etc/iot/certs/private.key,ca-path=/etc/iot/certs/AmazonCA.pem,role-aliases=GreengrassKVSRoleAlias\\\"\"}"
            }
        }
    }
}
EOF

aws greengrassv2 create-deployment \
    --cli-input-json file:///tmp/kvs-combined-deployment.json \
    --region $REGION
```

---

## Step 5-3: 両コンポーネントの起動確認

```bash
# 両方が RUNNING になるまで待つ
watch -n 5 sudo /greengrass/v2/bin/greengrass-cli component list
```

以下の状態になれば OK です。

```
Component Name: aws.greengrass.KinesisVideoStreamsWebRTC
    State: RUNNING

Component Name: aws.greengrass.KinesisVideoStreams
    State: RUNNING
```

---

## Step 5-4: 同時動作の確認

**WebRTC（ライブ視聴）の確認**

ブラウザから Viewer 接続します。

```
https://awslabs.github.io/amazon-kinesis-video-streams-webrtc-sdk-js/examples/index.html
```

| 項目 | 値 |
|---|---|
| Region | ap-northeast-1 |
| Channel Name | MyWSLCamera |
| Mode | Viewer |

**KVS Streams（録画）の確認**

```bash
# HLS URL 取得
DATA_ENDPOINT=$(aws kinesisvideo get-data-endpoint \
    --stream-name MyWSLCameraStream \
    --api-name GET_HLS_STREAMING_SESSION_URL \
    --region $REGION \
    --query DataEndpoint --output text)

aws kinesis-video-archived-media get-hls-streaming-session-url \
    --stream-name MyWSLCameraStream \
    --playback-mode LIVE \
    --endpoint-url $DATA_ENDPOINT \
    --region $REGION \
    --query HLSStreamingSessionURL --output text
```

取得した URL を VLC または `https://hls-js.netlify.app/demo/` で再生します。

---

## Step 5-5: リソース使用量の確認

WSL2 上で2つのコンポーネントが同時動作する場合のリソースを確認します。

```bash
# CPU / メモリ使用量
top -b -n 1 | grep -E "kvs|greengrass|java"

# より詳細に確認
ps aux | grep -E "kvs|greengrass|java" | \
    awk '{printf "%-50s CPU:%s%% MEM:%s%%\n", $11, $3, $4}'

# WSL2 全体のメモリ使用量
free -h
```

---

## Step 5-6: ログの同時監視

別ターミナルを2つ開いて両方のログを監視します。

```bash
# ターミナル 1: WebRTC ログ
tail -f /greengrass/v2/logs/aws.greengrass.KinesisVideoStreamsWebRTC.log

# ターミナル 2: Streams ログ
tail -f /greengrass/v2/logs/aws.greengrass.KinesisVideoStreams.log
```

両方のログでエラーがなく映像が流れていれば Step 5 完了です。

---

## 確認ポイントまとめ

| 確認項目 | 期待する結果 |
|---|---|
| 両コンポーネントの状態 | RUNNING |
| WebRTC Viewer | ブラウザでテストパターンが見える |
| KVS Streams HLS | VLC / hls.js でテストパターンが見える |
| CPU 使用率 | 2コンポーネント合計で安定している |
| ログにエラーなし | 両方のログでエラーが出ていない |

問題が起きた場合はどちらのログにエラーが出ているか教えていただければ確認します。

まず Step 1 の Greengrass インストールから始めますか？Jetson と WSL2 どちらで先に進めますか？

では Step 6「カスタムコンポーネント（KVSSessionManager）の動作確認」に進みます。

---

## Step 6-1: 作業ディレクトリの準備

```bash
mkdir -p ~/kvs-session-manager/artifacts
mkdir -p ~/kvs-session-manager/recipes
cd ~/kvs-session-manager
```

---

## Step 6-2: Python 依存パッケージのインストール

```bash
pip3 install \
    awsiotsdk \
    requests
```

---

## Step 6-3: コンポーネント本体の作成

```bash
cat > ~/kvs-session-manager/artifacts/component_main.py << 'EOF'
import os
import time
import json
import threading
import subprocess
import requests
import awsiot.greengrasscoreipc as greengrasscoreipc
from awsiot.greengrasscoreipc.model import (
    SubscribeToIoTCoreRequest,
    RestartComponentRequest,
    QOS
)

# -----------------------------------------------
# 設定
# -----------------------------------------------
CERTS_DIR = os.environ.get("CERTS_DIR", "/etc/iot/certs")
REGION = os.environ.get("AWS_REGION", "ap-northeast-1")
ROLE_ALIAS = os.environ.get("ROLE_ALIAS", "GreengrassKVSRoleAlias")
CREDENTIAL_ENDPOINT = os.environ.get("CREDENTIAL_ENDPOINT", "")
THING_NAME = os.environ.get("AWS_IOT_THING_NAME", "")
SESSION_REFRESH_MINUTES = int(os.environ.get("SESSION_REFRESH_MINUTES", "55"))

ipc_client = greengrasscoreipc.connect()


# -----------------------------------------------
# 一時認証情報の有効期限確認（デバッグ用）
# -----------------------------------------------
def get_credential_expiration():
    try:
        url = f"https://{CREDENTIAL_ENDPOINT}/role-aliases/{ROLE_ALIAS}/credentials"
        response = requests.get(
            url,
            cert=(
                f"{CERTS_DIR}/device.pem",
                f"{CERTS_DIR}/private.key"
            ),
            verify=f"{CERTS_DIR}/AmazonCA.pem",
            timeout=10
        )
        creds = response.json().get("credentials", {})
        expiration = creds.get("expiration", "unknown")
        print(f"[Credentials] Expiration: {expiration}")
        return expiration
    except Exception as e:
        print(f"[Credentials] Failed to get credentials: {e}")
        return None


# -----------------------------------------------
# コンポーネント再起動
# -----------------------------------------------
def restart_component(component_name: str):
    try:
        request = RestartComponentRequest(
            component_name=component_name
        )
        ipc_client.new_restart_component().activate(request).result(timeout=10)
        print(f"[Restart] {component_name} restarted successfully")
        return True
    except Exception as e:
        print(f"[Restart] Failed to restart {component_name}: {e}")
        return False


# -----------------------------------------------
# WebRTC セッション更新（定期実行）
# -----------------------------------------------
def refresh_webrtc_session():
    print(f"[SessionRefresh] Starting. Interval: {SESSION_REFRESH_MINUTES} minutes")

    while True:
        time.sleep(SESSION_REFRESH_MINUTES * 60)

        print("[SessionRefresh] Refreshing WebRTC session...")

        # 更新前の認証情報有効期限を確認
        get_credential_expiration()

        # WebRTC コンポーネントを再起動
        if restart_component("aws.greengrass.KinesisVideoStreamsWebRTC"):
            print("[SessionRefresh] WebRTC session refreshed successfully")
        else:
            print("[SessionRefresh] WebRTC session refresh failed")


# -----------------------------------------------
# 証明書更新ジョブの処理
# -----------------------------------------------
def on_cert_rotation_job(job_document: dict):
    print("[CertRotation] Starting certificate rotation...")

    new_cert_pem = job_document.get("certificatePem")
    new_key_pem = job_document.get("privateKey")

    if not new_cert_pem or not new_key_pem:
        print("[CertRotation] Missing certificate or key in job document")
        return

    try:
        # 証明書ファイルを上書き
        with open(f"{CERTS_DIR}/device.pem", "w") as f:
            f.write(new_cert_pem)
        with open(f"{CERTS_DIR}/private.key", "w") as f:
            f.write(new_key_pem)

        # パーミッション再設定
        for filepath in [
            f"{CERTS_DIR}/device.pem",
            f"{CERTS_DIR}/private.key"
        ]:
            subprocess.run(
                ["chown", "root:ggc_group", filepath],
                check=True
            )
            subprocess.run(
                ["chmod", "640", filepath],
                check=True
            )

        print("[CertRotation] Certificate files updated")

        # KVS 両コンポーネントを再起動
        for component in [
            "aws.greengrass.KinesisVideoStreams",
            "aws.greengrass.KinesisVideoStreamsWebRTC"
        ]:
            restart_component(component)
            time.sleep(3)  # 再起動間隔を少し空ける

        print("[CertRotation] Certificate rotation complete")

    except Exception as e:
        print(f"[CertRotation] Failed: {e}")


# -----------------------------------------------
# IoT Jobs サブスクライブ
# -----------------------------------------------
def subscribe_jobs():
    topic = f"$aws/things/{THING_NAME}/jobs/notify-next"

    class JobHandler:
        def on_stream_event(self, event):
            try:
                payload = json.loads(
                    event.message.payload.decode("utf-8")
                )
                execution = payload.get("execution", {})
                job_doc = execution.get("jobDocument", {})
                operation = job_doc.get("operation")

                print(f"[Jobs] Received job: {operation}")

                if operation == "CERT_ROTATION":
                    on_cert_rotation_job(job_doc)
                else:
                    print(f"[Jobs] Unknown operation: {operation}")

            except Exception as e:
                print(f"[Jobs] Error processing job: {e}")

        def on_stream_error(self, error):
            print(f"[Jobs] Stream error: {error}")

        def on_stream_closed(self):
            print("[Jobs] Stream closed, resubscribing...")
            time.sleep(5)
            subscribe_jobs()

    request = SubscribeToIoTCoreRequest(
        topic_name=topic,
        qos=QOS.AT_LEAST_ONCE
    )

    operation = ipc_client.new_subscribe_to_iot_core(JobHandler())
    operation.activate(request).result(timeout=10)
    print(f"[Jobs] Subscribed to {topic}")


# -----------------------------------------------
# エントリポイント
# -----------------------------------------------
if __name__ == "__main__":
    print("[Main] KVSSessionManager starting...")
    print(f"[Main] Thing: {THING_NAME}")
    print(f"[Main] Region: {REGION}")
    print(f"[Main] CertsDir: {CERTS_DIR}")
    print(f"[Main] SessionRefreshInterval: {SESSION_REFRESH_MINUTES} minutes")

    # 起動時に認証情報の有効期限を確認
    get_credential_expiration()

    # IoT Jobs サブスクライブ
    subscribe_jobs()

    # WebRTC セッション更新スレッド起動
    session_thread = threading.Thread(
        target=refresh_webrtc_session,
        daemon=True
    )
    session_thread.start()

    print("[Main] Component running...")

    # メインスレッド常駐
    while True:
        time.sleep(60)
        print("[Main] Heartbeat - component alive")
EOF
```

---

## Step 6-4: レシピの作成

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=ap-northeast-1
IOT_CRED_ENDPOINT=$(aws iot describe-endpoint \
    --endpoint-type iot:CredentialProvider \
    --region $REGION \
    --query endpointAddress --output text)

cat > ~/kvs-session-manager/recipes/recipe.yaml << EOF
---
RecipeFormatVersion: "2020-01-25"
ComponentName: com.example.KVSSessionManager
ComponentVersion: "1.0.0"
ComponentDescription: "WebRTCセッション更新とデバイス証明書更新を管理するコンポーネント"
ComponentPublisher: "MyCompany"

ComponentDependencies:
  aws.greengrass.Nucleus:
    VersionRequirement: ">=2.9.0"
    DependencyType: HARD
  aws.greengrass.KinesisVideoStreamsWebRTC:
    VersionRequirement: ">=2.0.0"
    DependencyType: SOFT
  aws.greengrass.KinesisVideoStreams:
    VersionRequirement: ">=7.0.0"
    DependencyType: SOFT

ComponentConfiguration:
  DefaultConfiguration:
    CertsDir: "/etc/iot/certs"
    Region: "${REGION}"
    RoleAlias: "GreengrassKVSRoleAlias"
    CredentialEndpoint: "${IOT_CRED_ENDPOINT}"
    SessionRefreshMinutes: "55"

Manifests:
  - Platform:
      os: linux
    Lifecycle:
      Run:
        RequiresPrivilege: true
        Script: >-
          CERTS_DIR={configuration:/CertsDir}
          AWS_REGION={configuration:/Region}
          ROLE_ALIAS={configuration:/RoleAlias}
          CREDENTIAL_ENDPOINT={configuration:/CredentialEndpoint}
          SESSION_REFRESH_MINUTES={configuration:/SessionRefreshMinutes}
          AWS_IOT_THING_NAME={iot:thingName}
          python3 {artifacts:path}/component_main.py
    Artifacts:
      - URI: "s3://YOUR_BUCKET/kvs-session-manager/component_main.py"
EOF
```

---

## Step 6-5: S3 バケットへアーティファクトをアップロード

```bash
BUCKET_NAME="greengrass-artifacts-${ACCOUNT_ID}-${REGION}"

# バケット作成
aws s3 mb s3://${BUCKET_NAME} --region $REGION

# アーティファクトをアップロード
aws s3 cp ~/kvs-session-manager/artifacts/component_main.py \
    s3://${BUCKET_NAME}/kvs-session-manager/component_main.py

# レシピの S3 URI を更新
sed -i "s|YOUR_BUCKET|${BUCKET_NAME}|g" \
    ~/kvs-session-manager/recipes/recipe.yaml

# 確認
cat ~/kvs-session-manager/recipes/recipe.yaml
```

---

## Step 6-6: Greengrass に S3 アクセス権限を付与

```bash
# Greengrass の TokenExchangeRole に S3 読み取り権限を追加
TOKEN_EXCHANGE_ROLE=$(aws iam list-roles \
    --query 'Roles[?contains(RoleName, `GreengrassV2TokenExchangeRole`)].RoleName' \
    --output text)

echo "TokenExchangeRole: ${TOKEN_EXCHANGE_ROLE}"

aws iam put-role-policy \
    --role-name ${TOKEN_EXCHANGE_ROLE} \
    --policy-name S3ArtifactAccess \
    --policy-document "{
        \"Version\": \"2012-10-17\",
        \"Statement\": [{
            \"Effect\": \"Allow\",
            \"Action\": [
                \"s3:GetObject\"
            ],
            \"Resource\": \"arn:aws:s3:::${BUCKET_NAME}/*\"
        }]
    }"
```

---

## Step 6-7: ローカルデプロイでの動作確認

本番デプロイ前にローカルで動作確認します。

```bash
# ローカルデプロイ用ディレクトリに配置
sudo mkdir -p /greengrass/v2/packages/artifacts/com.example.KVSSessionManager/1.0.0
sudo cp ~/kvs-session-manager/artifacts/component_main.py \
    /greengrass/v2/packages/artifacts/com.example.KVSSessionManager/1.0.0/

# ローカルデプロイ実行
sudo /greengrass/v2/bin/greengrass-cli deployment create \
    --recipeDir ~/kvs-session-manager/recipes \
    --artifactDir ~/kvs-session-manager/artifacts \
    --merge "com.example.KVSSessionManager=1.0.0"
```

---

## Step 6-8: 動作確認

```bash
# コンポーネント状態確認
sudo /greengrass/v2/bin/greengrass-cli component list

# ログ確認
tail -f /greengrass/v2/logs/com.example.KVSSessionManager.log
```

以下のようなログが出れば正常起動です。

```
[Main] KVSSessionManager starting...
[Main] Thing: robot-XXXX
[Main] Region: ap-northeast-1
[Credentials] Expiration: 2024-XX-XXTXX:XX:XXZ
[Jobs] Subscribed to $aws/things/robot-XXXX/jobs/notify-next
[Main] Component running...
[Main] Heartbeat - component alive
```

---

## Step 6-9: セッション更新の動作確認

本番では 55 分待つ必要がありますが、検証用に短い間隔で確認します。

```bash
# SessionRefreshMinutes を 2 分に変更してローカル再デプロイ
sudo /greengrass/v2/bin/greengrass-cli deployment create \
    --recipeDir ~/kvs-session-manager/recipes \
    --artifactDir ~/kvs-session-manager/artifacts \
    --merge "com.example.KVSSessionManager=1.0.0" \
    --update-config '{"com.example.KVSSessionManager":{"MERGE":{"SessionRefreshMinutes":"2"}}}'

# ログで2分後に再起動されることを確認
tail -f /greengrass/v2/logs/com.example.KVSSessionManager.log
```

2分後に以下のログが出れば WebRTC セッション更新が正常動作しています。

```
[SessionRefresh] Refreshing WebRTC session...
[Credentials] Expiration: 2024-XX-XXTXX:XX:XXZ
[Restart] aws.greengrass.KinesisVideoStreamsWebRTC restarted successfully
[SessionRefresh] WebRTC session refreshed successfully
```

---

## Step 6-10: 証明書更新ジョブのテスト

```bash
THING_NAME=$(sudo grep thingName /greengrass/v2/config/config.yaml \
    | awk '{print $2}' | tr -d '"')

# テスト用ジョブを発行
aws iot create-job \
    --job-id "cert-rotation-test-$(date +%s)" \
    --targets "arn:aws:iot:${REGION}:${ACCOUNT_ID}:thing/${THING_NAME}" \
    --document '{
        "operation": "CERT_ROTATION",
        "certificatePem": "-----BEGIN CERTIFICATE-----\nTEST\n-----END CERTIFICATE-----",
        "privateKey": "-----BEGIN RSA PRIVATE KEY-----\nTEST\n-----END RSA PRIVATE KEY-----"
    }' \
    --region $REGION
```

ログで証明書更新処理が走ることを確認します。

```bash
tail -f /greengrass/v2/logs/com.example.KVSSessionManager.log
```

---

## 全 Step 完了後の構成まとめ

```
Greengrass Core Device（WSL2）
  ├─ aws.greengrass.Nucleus          : MQTT / 認証情報管理
  ├─ aws.greengrass.KinesisVideoStreamsWebRTC : WebRTC ライブ配信
  ├─ aws.greengrass.KinesisVideoStreams       : KVS 録画保存
  └─ com.example.KVSSessionManager           : セッション更新 / 証明書更新
```

Step 6-7 のローカルデプロイあたりで詰まることが多いので、ログを確認しながら進めてください。問題があればログを貼っていただければ確認します。

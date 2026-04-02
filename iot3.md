順番に整理します。大きく5つのステップです。

---

## ステップ全体像

```
① AWSの事前準備（1回だけ）
② クレーム証明書の作成
③ Fleet Provisioning の設定
④ Certificate Provider の設定
⑤ デバイス側の実装
```

---

## ステップ①：AWSの事前準備

### IAMロールの作成

```bash
# Fleet Provisioning用のIAMロール
aws iam create-role \
  --role-name IoTFleetProvisioningRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "iot.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name IoTFleetProvisioningRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSIoTThingsRegistration
```

### Thing Groupの作成

```bash
# 3環境分のThing Groupを作成
aws iot create-thing-group --thing-group-name robots-dev
aws iot create-thing-group --thing-group-name robots-stg
aws iot create-thing-group --thing-group-name robots-prod
```

### IoTポリシーの作成

```bash
# dev用ポリシー
aws iot create-policy \
  --policy-name RobotPolicy-dev \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "iot:Connect",
        "Resource": "arn:aws:iot:ap-northeast-1:123456789:client/${iot:Connection.Thing.ThingName}"
      },
      {
        "Effect": "Allow",
        "Action": ["iot:Publish", "iot:Subscribe", "iot:Receive"],
        "Resource": [
          "arn:aws:iot:ap-northeast-1:123456789:topic/dev/${iot:Connection.Thing.ThingName}/*",
          "arn:aws:iot:ap-northeast-1:123456789:topicfilter/dev/${iot:Connection.Thing.ThingName}/*",
          "arn:aws:iot:ap-northeast-1:123456789:topic/management/topic/${iot:Connection.Thing.ThingName}/*",
          "arn:aws:iot:ap-northeast-1:123456789:topicfilter/management/topic/${iot:Connection.Thing.ThingName}/*"
        ]
      }
    ]
  }'

# stg / prod も同様に作成（dev部分を置き換えるだけ）
```

---

## ステップ②：擬似プライベートCAとクレーム証明書の作成

### 擬似CAの作成

```bash
# CA秘密鍵を生成
openssl genrsa -out ca.key 4096

# 自己署名CA証明書を生成（有効期限10年）
openssl req -x509 -new -nodes \
  -key ca.key \
  -sha256 \
  -days 3650 \
  -out ca.crt \
  -subj "/CN=RobotCA/O=YourCompany/C=JP"

# Secret Managerに保存
aws secretsmanager create-secret \
  --name "iot/ca" \
  --secret-string "{
    \"certificate\": \"$(cat ca.crt | sed 's/$/\\n/' | tr -d '\n')\",
    \"private_key\": \"$(cat ca.key | sed 's/$/\\n/' | tr -d '\n')\"
  }"

# ローカルのCA秘密鍵を削除
rm ca.key ca.crt
```

### クレーム証明書の作成

```bash
# IoT Coreでクレーム証明書を作成
aws iot create-keys-and-certificate \
  --set-as-active \
  --certificate-pem-outfile claim.crt \
  --public-key-outfile claim.pub \
  --private-key-outfile claim.key

# クレーム証明書用ポリシーを作成（Fleet Provisioningトピックのみ許可）
aws iot create-policy \
  --policy-name ClaimPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "iot:Connect",
        "Resource": "*"
      },
      {
        "Effect": "Allow",
        "Action": ["iot:Publish", "iot:Subscribe", "iot:Receive"],
        "Resource": [
          "arn:aws:iot:ap-northeast-1:123456789:topic/$aws/certificates/create-from-csr/*",
          "arn:aws:iot:ap-northeast-1:123456789:topicfilter/$aws/certificates/create-from-csr/*",
          "arn:aws:iot:ap-northeast-1:123456789:topic/$aws/provisioning-templates/RobotProvisioningTemplate/provision/*",
          "arn:aws:iot:ap-northeast-1:123456789:topicfilter/$aws/provisioning-templates/RobotProvisioningTemplate/provision/*"
        ]
      }
    ]
  }'

# ポリシーをクレーム証明書に付与
CERT_ARN=$(aws iot list-certificates --query 'certificates[0].certificateArn' --output text)
aws iot attach-policy \
  --policy-name ClaimPolicy \
  --target $CERT_ARN
```

---

## ステップ③：Fleet Provisioningの設定

### Pre-Provisioning Hook Lambdaの作成

```python
# pre_provisioning_hook.py
import boto3

iot = boto3.client('iot')

def lambda_handler(event, context):
    serial    = event['parameters']['SerialNumber']
    environment = event['parameters']['Environment']

    # 有効な環境かチェック
    if environment not in ['dev', 'stg', 'prod']:
        return {'allowProvisioning': False}

    # 重複登録チェック
    try:
        iot.describe_thing(thingName=serial)
        principals = iot.list_thing_principals(thingName=serial)
        if len(principals['principals']) > 0:
            return {'allowProvisioning': False}
    except iot.exceptions.ResourceNotFoundException:
        pass

    return {
        'allowProvisioning': True,
        'parameterOverrides': {
            'SerialNumber': serial,
            'Environment': environment
        }
    }
```

```bash
# Lambdaをデプロイ
zip hook.zip pre_provisioning_hook.py

aws lambda create-function \
  --function-name PreProvisioningHook \
  --runtime python3.11 \
  --handler pre_provisioning_hook.lambda_handler \
  --role arn:aws:iam::123456789:role/LambdaIoTRole \
  --zip-file fileb://hook.zip

# IoT CoreからLambdaを呼び出す権限を付与
aws lambda add-permission \
  --function-name PreProvisioningHook \
  --statement-id iot-invoke \
  --action lambda:InvokeFunction \
  --principal iot.amazonaws.com
```

### Fleet Provisioningテンプレートの作成

```bash
aws iot create-provisioning-template \
  --template-name RobotProvisioningTemplate \
  --provisioning-role-arn arn:aws:iam::123456789:role/IoTFleetProvisioningRole \
  --pre-provisioning-hook '{
    "targetArn": "arn:aws:lambda:ap-northeast-1:123456789:function:PreProvisioningHook"
  }' \
  --template-body '{
    "Parameters": {
      "SerialNumber": {"Type": "String"},
      "Environment":  {"Type": "String"},
      "AWS::IoT::Certificate::Id": {"Type": "String"}
    },
    "Resources": {
      "thing": {
        "Type": "AWS::IoT::Thing",
        "Properties": {
          "ThingName": {"Ref": "SerialNumber"},
          "ThingGroups": [
            {"Fn::Join": ["-", ["robots", {"Ref": "Environment"}]]}
          ],
          "AttributePayload": {
            "environment": {"Ref": "Environment"}
          }
        }
      },
      "certificate": {
        "Type": "AWS::IoT::Certificate",
        "Properties": {
          "CertificateId": {"Ref": "AWS::IoT::Certificate::Id"},
          "Status": "Active"
        }
      },
      "policy": {
        "Type": "AWS::IoT::Policy",
        "Properties": {
          "PolicyName": {
            "Fn::Join": ["-", ["RobotPolicy", {"Ref": "Environment"}]]
          }
        }
      }
    }
  }' \
  --enabled
```

---

## ステップ④：Certificate Providerの設定

```python
# certificate_provider.py
import boto3
import json
from cryptography import x509
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives.asymmetric import rsa
import datetime

sm = boto3.client('secretsmanager')

def lambda_handler(event, context):
    csr_pem = event['certificateSigningRequest']

    # ① CSR検証
    csr = verify_csr(csr_pem)

    # ② Secret ManagerからCA取得
    ca_secret = json.loads(
        sm.get_secret_value(SecretId='iot/ca')['SecretString']
    )
    ca_cert = x509.load_pem_x509_certificate(
        ca_secret['certificate'].encode(),
        default_backend()
    )
    ca_key = serialization.load_pem_private_key(
        ca_secret['private_key'].encode(),
        password=None,
        backend=default_backend()
    )

    # ③ 証明書を発行（有効期限395日）
    cert = (
        x509.CertificateBuilder()
        .subject_name(csr.subject)
        .issuer_name(ca_cert.subject)
        .public_key(csr.public_key())
        .serial_number(x509.random_serial_number())
        .not_valid_before(datetime.datetime.utcnow())
        .not_valid_after(
            datetime.datetime.utcnow() + datetime.timedelta(days=395)
        )
        .sign(ca_key, hashes.SHA256())
    )

    return {
        'certificatePem': cert.public_bytes(
            serialization.Encoding.PEM
        ).decode()
    }


def verify_csr(csr_pem):
    csr = x509.load_pem_x509_csr(
        csr_pem.encode(),
        default_backend()
    )
    if not csr.is_signature_valid:
        raise ValueError("CSR署名が不正です")

    public_key = csr.public_key()
    if isinstance(public_key, rsa.RSAPublicKey):
        if public_key.key_size < 2048:
            raise ValueError("鍵長が不十分です")

    return csr
```

```bash
# Certificate Provider Lambdaをデプロイ
zip cert_provider.zip certificate_provider.py

aws lambda create-function \
  --function-name CertificateProvider \
  --runtime python3.11 \
  --handler certificate_provider.lambda_handler \
  --role arn:aws:iam::123456789:role/LambdaIoTRole \
  --zip-file fileb://cert_provider.zip

# Certificate Providerとして登録
aws iot create-certificate-provider \
  --certificate-provider-name RobotCertProvider \
  --lambda-function-arn arn:aws:lambda:ap-northeast-1:123456789:function:CertificateProvider \
  --account-default-for-operations CREATE_CERTIFICATE_FROM_CSR
```

---

## ステップ⑤：デバイス側の実装

### ディレクトリ構成

```
/etc/iot/
  ├─ certs/
  │    ├─ claim.crt       ← 工場で焼き込み
  │    ├─ claim.key       ← 工場で焼き込み
  │    ├─ AmazonRootCA1.pem ← 工場で焼き込み
  │    ├─ device.crt      ← Provisioning後に生成
  │    └─ device.key      ← Provisioning後に生成
  └─ config.json          ← 工場で焼き込み
```

### config.json

```json
{
  "iot_endpoint": "xxxxx-ats.iot.ap-northeast-1.amazonaws.com",
  "template_name": "RobotProvisioningTemplate",
  "environment": "dev"
}
```

### entrypoint.sh

```bash
#!/bin/sh

CERT_PATH="/etc/iot/certs/device.crt"

if [ ! -f "$CERT_PATH" ]; then
    echo "証明書なし → Fleet Provisioning実行"
    python3 /app/provisioning.py

    if [ $? -ne 0 ]; then
        echo "Provisioning失敗"
        exit 1
    fi
fi

echo "通常起動"
exec python3 /app/iot_client.py
```

### provisioning.py

```python
import json
import os
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization, hashes
from cryptography.hazmat.backends import default_backend
import cryptography.x509 as x509
from cryptography.x509.oid import NameOID
from awscrt import mqtt
from awsiot import mqtt_connection_builder

CERT_DIR    = "/etc/iot/certs"
CONFIG_PATH = "/etc/iot/config.json"

def get_device_id():
    with open('/proc/cpuinfo') as f:
        for line in f:
            if 'Serial' in line:
                return line.split(':')[1].strip()

def run():
    with open(CONFIG_PATH) as f:
        config = json.load(f)

    device_id   = get_device_id()
    environment = config['environment']

    # ① 秘密鍵・CSRをデバイスで生成
    private_key = rsa.generate_private_key(
        public_exponent=65537,
        key_size=2048,
        backend=default_backend()
    )
    csr = x509.CertificateSigningRequestBuilder().subject_name(
        x509.Name([x509.NameAttribute(NameOID.COMMON_NAME, device_id)])
    ).sign(private_key, hashes.SHA256(), default_backend())

    csr_pem = csr.public_bytes(serialization.Encoding.PEM).decode()

    # ② クレーム証明書でIoT Coreに接続
    mqtt_conn = mqtt_connection_builder.mtls_from_path(
        endpoint=config['iot_endpoint'],
        cert_filepath=f"{CERT_DIR}/claim.crt",
        pri_key_filepath=f"{CERT_DIR}/claim.key",
        ca_filepath=f"{CERT_DIR}/AmazonRootCA1.pem",
        client_id=f"provisioning-{device_id}"
    )
    mqtt_conn.connect().result()

    # ③ CSRから証明書を発行（Certificate Providerが署名）
    cert_pem, cert_id, ownership_token = create_cert_from_csr(
        mqtt_conn, csr_pem
    )

    # ④ Fleet Provisioningテンプレートを実行
    register_thing(
        mqtt_conn,
        config['template_name'],
        ownership_token,
        device_id,
        environment
    )

    # ⑤ 証明書・秘密鍵をホスト側に保存
    with open(f"{CERT_DIR}/device.crt", 'w') as f:
        f.write(cert_pem)

    with open(f"{CERT_DIR}/device.key", 'wb') as f:
        f.write(private_key.private_bytes(
            serialization.Encoding.PEM,
            serialization.PrivateFormat.TraditionalOpenSSL,
            serialization.NoEncryption()
        ))

    mqtt_conn.disconnect().result()

    # ⑥ クレーム証明書を削除
    os.remove(f"{CERT_DIR}/claim.crt")
    os.remove(f"{CERT_DIR}/claim.key")

    print(f"Provisioning完了: {device_id} / {environment}")
```

### docker-compose.yml

```yaml
services:
  iot-client:
    image: your-iot-client
    volumes:
      - /etc/iot:/etc/iot
    entrypoint: ["/bin/sh", "/app/entrypoint.sh"]
    restart: unless-stopped
```

---

## 手順まとめ

```
① AWSの事前準備
   └─ IAMロール・Thing Group・IoTポリシーを作成

② 擬似CA・クレーム証明書の作成
   └─ openssl でCA作成 → Secret Managerに保存
   └─ クレーム証明書を作成・ClaimPolicyを付与

③ Fleet Provisioningの設定
   └─ Pre-Provisioning Hook Lambdaをデプロイ
   └─ Provisioningテンプレートを作成

④ Certificate Providerの設定
   └─ Certificate Provider Lambdaをデプロイ
   └─ IoT CoreにCertificate Providerとして登録

⑤ デバイス側の実装
   └─ claim.crt + config.json を焼き込み
   └─ provisioning.py + iot_client.py を実装
   └─ docker-compose.yml を設定
```

どのステップから詳しく見ていきますか？

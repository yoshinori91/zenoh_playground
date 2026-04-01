まず全体のデータフローを図示して、その後各ポイントを解説します。各要素を順番に解説します。

---

## ① Defender → SNS の設定

Defender の Audit 通知を SNS に飛ばすには、IoT コンソールの `Defend > Audit > Settings` から SNS トピックを指定します。CLI でも設定できます。

```bash
aws iot update-account-audit-configuration \
  --audit-notification-target-configurations '{
    "SNS": {
      "targetArn": "arn:aws:sns:ap-northeast-1:123456789:iot-defender-audit",
      "roleArn":   "arn:aws:iam::123456789:role/IoTDefenderSNSRole",
      "enabled":   true
    }
  }'
```

---

## ② SNS → Lambda のサブスクリプション

SNS トピックに Lambda を直接サブスクライブするだけです。EventBridge は不要。

```bash
aws sns subscribe \
  --topic-arn "arn:aws:sns:ap-northeast-1:123456789:iot-defender-audit" \
  --protocol lambda \
  --notification-endpoint "arn:aws:lambda:ap-northeast-1:123456789:function:cert-rotation"
```

Lambda 側にリソースベースポリシーも必要です。

```bash
aws lambda add-permission \
  --function-name cert-rotation \
  --statement-id sns-invoke \
  --action lambda:InvokeFunction \
  --principal sns.amazonaws.com \
  --source-arn "arn:aws:sns:ap-northeast-1:123456789:iot-defender-audit"
```

---

## ③ Lambda のペイロード処理

SNS から Lambda に渡るイベントは `Records[0].Sns.Message` の中に JSON 文字列で入っています。

```python
import json, boto3

iot = boto3.client('iot')

def lambda_handler(event, context):
    msg = json.loads(event['Records'][0]['Sns']['Message'])

    # 非準拠がなければ即終了
    if msg.get('nonCompliantChecksCount', 0) == 0:
        return

    # taskId で期限切れ証明書の一覧を取得
    findings = iot.list_audit_findings(
        checkName='DEVICE_CERTIFICATE_EXPIRING_CHECK',
        taskId=msg['taskId']
    )

    for finding in findings['findings']:
        cert_id = finding['nonCompliantResource']['resourceIdentifier']['deviceCertificateId']
        # 紐づく Thing (clientId) を特定してローテーション処理へ
        rotate_certificate(cert_id)
```

---

## ④ エラーハンドリング（重要）

SNS → Lambda の直結構成では、Lambda の失敗時に SNS が**2回まで自動リトライ**します。それでも失敗した場合に備えて、Lambda に DLQ（SQS）を設定しておくことを推奨します。

```bash
aws lambda update-function-event-invoke-config \
  --function-name cert-rotation \
  --maximum-retry-attempts 2 \
  --destination-config '{
    "OnFailure": {
      "Destination": "arn:aws:sqs:ap-northeast-1:123456789:cert-rotation-dlq"
    }
  }'
```

---

## ⑤ 必要な IAM 権限まとめ

Lambda の実行ロールに付与すべき権限です。

```json
{
  "Effect": "Allow",
  "Action": [
    "iot:ListAuditFindings",
    "iot:ListThingPrincipals",
    "iot:DescribeCertificate",
    "iot:CreateCertificateFromCsr",
    "iot:AttachPolicy",
    "iot:UpdateCertificate",
    "iot:DeleteCertificate",
    "iot:Publish"
  ],
  "Resource": "*"
}
```

ノード（各ボックス）をクリックすると追加で聞けるようにしてあります。どこか掘り下げたい部分はありますか？

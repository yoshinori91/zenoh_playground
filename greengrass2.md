なるほど、整理するとカスタムコンポーネントで Component SDK（IPC）を使う理由が2つあるわけですね。

---

## 2つの更新処理の違い

| 項目 | 周期 | 何を更新するか | 手段 |
|---|---|---|---|
| デバイス証明書 | 数ヶ月〜1年単位 | `/etc/iot/certs/` の証明書ファイル + Nucleus 再接続 | IoT Jobs + カスタムコンポーネント |
| WebRTC セッション | **1時間毎** | IoT Credential Provider の一時認証情報（STS トークン） | カスタムコンポーネントが定期的に再取得 |

WebRTC セッションが1時間で切れるのは、IoT Credential Provider が発行する **一時認証情報（STS）の有効期限がデフォルト1時間**だからです。この一時認証情報を使って KVS Signaling Channel に接続しているため、期限切れになると WebRTC の Master 接続が切断されます。

---

## カスタムコンポーネントの役割

```
カスタムコンポーネント（常駐）
  │
  ├─ 定期タスク（1時間毎）
  │    └─ IoT Credential Provider から新しい一時認証情報を取得
  │         └─ KVS WebRTC Component を再起動して新トークンで再接続
  │
  └─ イベントタスク（IoT Jobs 受信時）
       └─ デバイス証明書を更新
            └─ Nucleus 再接続 → KVS 両コンポーネント再起動
```

---

## 実装例

```python
# component_main.py
import time
import threading
import subprocess
import requests
import awsiot.greengrasscoreipc as greengrasscoreipc
from awsiot.greengrasscoreipc.model import (
    SubscribeToIoTCoreRequest,
    RestartComponentRequest,
    QOS
)

CERTS_DIR = "/etc/iot/certs"
REGION = "ap-northeast-1"
ROLE_ALIAS = "GreengrassKVSRoleAlias"
CREDENTIAL_ENDPOINT = "XXXX.credentials.iot.ap-northeast-1.amazonaws.com"

ipc_client = greengrasscoreipc.connect()


# -----------------------------------------------
# WebRTC セッション更新（1時間毎）
# -----------------------------------------------
def refresh_webrtc_session():
    """
    IoT Credential Provider から一時認証情報を再取得し、
    KVS WebRTC Component を再起動することでセッションを更新する
    """
    while True:
        # 55分毎に更新（期限切れ5分前）
        time.sleep(55 * 60)

        print("[SessionRefresh] Restarting KVS WebRTC component...")
        try:
            request = RestartComponentRequest(
                component_name="aws.greengrass.KinesisVideoStreamsWebRTC"
            )
            ipc_client.new_restart_component().activate(request).result(timeout=10)
            print("[SessionRefresh] KVS WebRTC restarted successfully")
        except Exception as e:
            print(f"[SessionRefresh] Failed to restart WebRTC: {e}")


# -----------------------------------------------
# 一時認証情報の取得確認（デバッグ用）
# -----------------------------------------------
def get_temp_credentials():
    """
    IoT Credential Provider から STS 一時認証情報を取得する
    KVS WebRTC SDK 内部でも同様の処理が行われているが、
    有効期限の監視用に明示的に取得して確認する
    """
    url = f"https://{CREDENTIAL_ENDPOINT}/role-aliases/{ROLE_ALIAS}/credentials"
    response = requests.get(
        url,
        cert=(
            f"{CERTS_DIR}/device.pem",
            f"{CERTS_DIR}/private.key"
        ),
        verify=f"{CERTS_DIR}/AmazonCA.pem"
    )
    creds = response.json()["credentials"]
    print(f"[Credentials] Expiration: {creds['expiration']}")
    return creds


# -----------------------------------------------
# デバイス証明書更新（IoT Jobs 受信時）
# -----------------------------------------------
def on_cert_rotation_job(job_payload: dict):
    """
    IoT Jobs から新証明書の S3 URL 等を受け取って更新する
    """
    print("[CertRotation] Starting certificate rotation...")

    new_cert_pem = job_payload.get("certificatePem")
    new_key_pem = job_payload.get("privateKey")

    # 証明書ファイルを上書き
    with open(f"{CERTS_DIR}/device.pem", "w") as f:
        f.write(new_cert_pem)
    with open(f"{CERTS_DIR}/private.key", "w") as f:
        f.write(new_key_pem)

    # パーミッション再設定
    subprocess.run(["chown", "root:ggc_group", f"{CERTS_DIR}/device.pem"])
    subprocess.run(["chmod", "640", f"{CERTS_DIR}/device.pem"])
    subprocess.run(["chown", "root:ggc_group", f"{CERTS_DIR}/private.key"])
    subprocess.run(["chmod", "640", f"{CERTS_DIR}/private.key"])

    # KVS 両コンポーネント再起動
    for component in [
        "aws.greengrass.KinesisVideoStreams",
        "aws.greengrass.KinesisVideoStreamsWebRTC"
    ]:
        request = RestartComponentRequest(component_name=component)
        ipc_client.new_restart_component().activate(request).result(timeout=10)
        print(f"[CertRotation] Restarted {component}")

    print("[CertRotation] Certificate rotation complete")


# -----------------------------------------------
# IoT Jobs サブスクライブ
# -----------------------------------------------
def subscribe_jobs():
    import os
    thing_name = os.environ.get("AWS_IOT_THING_NAME")
    topic = f"$aws/things/{thing_name}/jobs/notify-next"

    request = SubscribeToIoTCoreRequest(
        topic_name=topic,
        qos=QOS.AT_LEAST_ONCE
    )

    class JobHandler:
        def on_stream_event(self, event):
            payload = event.message.payload.decode("utf-8")
            import json
            job = json.loads(payload)
            if job.get("execution", {}).get("jobDocument", {}).get("operation") == "CERT_ROTATION":
                on_cert_rotation_job(job["execution"]["jobDocument"])

        def on_stream_error(self, error):
            print(f"[Jobs] Stream error: {error}")

        def on_stream_closed(self):
            print("[Jobs] Stream closed")

    operation = ipc_client.new_subscribe_to_iot_core(JobHandler())
    operation.activate(request).result(timeout=10)
    print(f"[Jobs] Subscribed to {topic}")


# -----------------------------------------------
# エントリポイント
# -----------------------------------------------
if __name__ == "__main__":
    # IoT Jobs サブスクライブ開始
    subscribe_jobs()

    # WebRTC セッション更新スレッド起動
    t = threading.Thread(target=refresh_webrtc_session, daemon=True)
    t.start()

    print("[Main] Component running...")

    # メインスレッドは常駐
    while True:
        time.sleep(60)
```

---

## レシピ

```yaml
---
RecipeFormatVersion: "2020-01-25"
ComponentName: com.example.KVSSessionManager
ComponentVersion: "1.0.0"
ComponentDescription: "WebRTCセッション更新とデバイス証明書更新を管理するコンポーネント"
ComponentPublisher: "MyCompany"

ComponentDependencies:
  aws.greengrass.Nucleus:
    VersionRequirement: ">=2.9.0"
  aws.greengrass.KinesisVideoStreamsWebRTC:
    VersionRequirement: ">=2.0.0"
    DependencyType: SOFT   # WebRTC が落ちても本コンポーネントは継続動作
  aws.greengrass.KinesisVideoStreams:
    VersionRequirement: ">=7.0.0"
    DependencyType: SOFT

ComponentConfiguration:
  DefaultConfiguration:
    CertsDir: "/etc/iot/certs"
    Region: "ap-northeast-1"
    RoleAlias: "GreengrassKVSRoleAlias"
    SessionRefreshIntervalMinutes: 55

Manifests:
  - Platform:
      os: linux
    Lifecycle:
      Run:
        RequiresPrivilege: true
        Script: "python3 {artifacts:path}/component_main.py"
    Artifacts:
      - URI: "s3://YOUR_BUCKET/component_main.py"
```

`DependencyType: SOFT` にしているのは、KVS コンポーネントを再起動する際に本コンポーネント自身が停止しないようにするためです。`HARD`（デフォルト）にすると依存先の再起動に引きずられて本コンポーネントも停止してしまいます。

---

## 処理の流れの整理

```
起動時
  └─ IoT Jobs トピックをサブスクライブ
  └─ WebRTC セッション更新スレッドを起動（55分おきにループ）

55分後
  └─ KVS WebRTC Component を RestartComponent（IPC）で再起動
       └─ 再起動後に SDK が新しい一時認証情報を自動取得して Signaling Channel に再接続

IoT Jobs 受信時
  └─ /etc/iot/certs/ の証明書を上書き
  └─ KVS WebRTC + KVS Streams 両方を RestartComponent で再起動
  └─ Nucleus は次回 MQTT 接続時に新証明書で再接続
```

WebRTC のセッション更新はコンポーネントの再起動で解決できる点がシンプルで、SDK 内部が再起動時に一時認証情報の再取得を自動で行ってくれるので明示的なトークン管理が不要です。

いいえ、`ComponentDependencies` はセッション管理とは関係ありません。

---

## `ComponentDependencies` の実際の役割

これは単純に**コンポーネントの起動順序と依存関係の宣言**です。

```yaml
ComponentDependencies:
  aws.greengrass.KinesisVideoStreamsWebRTC:
    VersionRequirement: ">=2.0.0"
    DependencyType: SOFT
```

これが行っているのは以下の3つだけです。

| 機能 | 説明 |
|---|---|
| 起動順序の保証 | 依存先コンポーネントが起動してから自コンポーネントが起動する |
| バージョン整合性チェック | 指定バージョン未満ならデプロイ時にエラーにする |
| SOFT/HARD の違い | HARD は依存先が落ちたら自分も停止、SOFT は自分は継続動作 |

セッション管理・証明書更新・再起動トリガーは一切行いません。

---

## セッション管理を実際に担っているのは何か

```
ComponentDependencies  →  起動順序の宣言だけ
                                    
IPC RestartComponent   →  ★ これが実際にセッション更新を行う
（コード内の処理）
```

先ほどのコードのこの部分が本体です。

```python
# ★ セッション更新の本体はここ
def refresh_webrtc_session():
    while True:
        time.sleep(55 * 60)  # 55分待つ

        # IPC 経由で WebRTC コンポーネントを再起動
        # → 再起動時に SDK が一時認証情報を再取得する
        request = RestartComponentRequest(
            component_name="aws.greengrass.KinesisVideoStreamsWebRTC"
        )
        ipc_client.new_restart_component().activate(request).result(timeout=10)
```

`RestartComponent` を IPC 経由で呼ぶことで KVS WebRTC Component が再起動し、その起動処理の中で SDK が IoT Credential Provider から新しい一時認証情報を自動取得します。`ComponentDependencies` はその再起動対象を事前に宣言しているだけです。

---

## 整理すると

```
ComponentDependencies の役割
  └─ 「このコンポーネントは KVS WebRTC より後に起動してね」という宣言

セッション管理の実体
  └─ コード内の threading.Thread で 55 分おきに
       RestartComponent（IPC）を呼ぶ処理
```

`ComponentDependencies` を書かなくてもセッション管理のコードは動きますが、KVS WebRTC が起動する前に本コンポーネントが動き出してしまうリスクがあるため、起動順序の保証として書いています。

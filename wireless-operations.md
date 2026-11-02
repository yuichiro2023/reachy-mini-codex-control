# Wireless：公式HTTP APIによる起床・収納・daemon停止

この文書は2026-11-02のWindows PowerShellによる実測と、そのときに使った公式APIのコマンドを公開用に整理したものです。`$base`はWireless本体の管理APIです。今回の成功経路ではSSHを使っていません。APIへの到達やdaemonの状態は再実行時に変わり得ます。

## 実測結果

| 段階 | APIで観測したこと | 人による物理確認 |
|---|---|---|
| 起床前 | daemon `running`、backend `ready=true`、errorなし。管理アプリなし、ロック`free`、実行中動作0件、モーター`disabled`、自動起動アプリ`null` | 起床できる空間を確保 |
| モーター有効化 | POSTがHTTP 200、後のGETで`enabled` | 急な動き・引っかかり・異音なし |
| wake-up | POSTがHTTP 200で動作UUIDを返し、その後の実行中動作は0件 | 頭・アンテナが通常姿勢に到達、異音・引っかかりなし |
| 収納・停止 | `stop?goto_sleep=true`の後、daemon `stopped`、error `null` | 頭・アンテナが収納姿勢に到達、異音・引っかかりなし |
| 08:53の再起動 | 停止中から`start?wake_up=false`の後、daemon `running`、backend ready、errorなし。モーター有効化後`enabled`、wake-up後の実行中動作0件 | 通常姿勢に到達、異常なし |
| 08:56の再停止 | `stop?goto_sleep=true`の後、daemon `stopped`、error `null` | 収納姿勢に到達、異常なし |

最新の08:53–08:56 JSTの連続操作は[2体の記録](pair-start-stop.md)にまとめています。非同期ジョブIDや動作UUIDは受付の識別子です。`/api/move/running`が0件になっても動作成功は証明できないため、実機の姿勢を別に確認しました。停止したのはdaemon内部で、物理電源は切っていません。

## 再現時のコマンドと判定

各GET・POSTは実機への操作です。実験では段階ごとにコマンドと影響を示し、個別承認を得ました。以下を一括実行せず、各結果と実機の状態を確認してから次へ進みます。

### 1. 接続・daemon状態の確認

本体の電源とWi-Fiを確認し、頭とアンテナの周囲を空けます。状態照会の承認後に実行したPowerShell例です。

```powershell
$base = 'http://reachy-mini.local:8000'
$daemon = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
$daemon | Select-Object state, error, camera_specs_name, backend_status
```

HTTPに届かない場合、API経由のdaemon起動はできません。名前解決・Wi-Fi・電源を確認します。`running`なら起動POSTを重ねません。

### 2. daemonが`stopped`の場合だけ起動

`wake_up=false`はdaemon内部を開始し、このPOST自体では起床動作をしません。`stopped`とAPI応答を事前確認し、起動の影響について承認を得てから実行します。08:53 JSTに、この分岐も実機で成功しました。

```powershell
$job = Invoke-RestMethod -Method Post -Uri "$base/api/daemon/start?wake_up=false" -TimeoutSec 10
$job.job_id
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5 |
    Select-Object state, error, backend_status
```

ジョブ受付だけを成功とせず、`running`、backend ready、errorなしを確認します。`starting`ならGETで経過を確認し、POSTを再送しません。

### 3. 起床前の競合・モーター状態の確認

daemonが正常に`running`であることを確認してから、アプリ、ロック、実行中動作、モーター、自動起動アプリを読みます。状態照会の承認が必要です。

```powershell
$app = Invoke-RestMethod -Method Get -Uri "$base/api/apps/current-app-status" -TimeoutSec 5
$lock = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/robot-app-lock-status" -TimeoutSec 5
$moves = Invoke-RestMethod -Method Get -Uri "$base/api/move/running" -TimeoutSec 5
$motors = Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
$startup = Invoke-RestMethod -Method Get -Uri "$base/api/apps/startup-app" -TimeoutSec 5
[pscustomobject]@{
    app_running = ($null -ne $app)
    lock_state = $lock.state
    running_move_count = $moves.Count
    motor_mode = $motors.mode
    startup_app = $startup.startup_app
} | Format-List
```

続行条件は管理アプリなし、ロック`free`、実行中動作0件、daemon/backend errorなしです。自動起動アプリがある場合は起床後に動く可能性があるため、ここで影響を検討します。管理APIで見えないクライアントが存在しないことまでは証明できません。

### 4. モーター有効化、続いてwake-up

モーターが`disabled`の場合のみ、トルク有効化の影響について個別承認を得てPOSTを1回送ります。急な動き、引っかかり、異音を監視します。

```powershell
Invoke-RestMethod -Method Post -Uri "$base/api/motors/set_mode/enabled" -TimeoutSec 8
Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
```

後のGETで`enabled`を確認し、異常がなければ起床動作について別の個別承認を得ます。頭とアンテナが動くため、周囲に人や物がないこと、動作0件、アプリなし、ロック`free`を再確認してから1回だけ送ります。

```powershell
$task = Invoke-RestMethod -Method Post -Uri "$base/api/move/play/wake_up" -TimeoutSec 10
$task.uuid
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$base/api/move/running" -TimeoutSec 5
```

最後に人が通常姿勢への到達と異常の有無を確認します。既に通常姿勢でモーターが有効ならwake-upを繰り返しません。タイムアウト・HTTP error・異常動作があっても、状態を確認する前にPOSTを再送しません。

### 5. 収納姿勢へ動かしてdaemonを停止

使用中のアプリや動作がないこと、ロック`free`、daemon/backend正常を再GETし、頭とアンテナの周囲を空けます。このPOSTは収納姿勢への物理移動とdaemon内部の停止を伴います。影響について個別承認を得てから実行します。

```powershell
$job = Invoke-RestMethod -Method Post -Uri "$base/api/daemon/stop?goto_sleep=true" -TimeoutSec 10
$job.job_id
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5 |
    Select-Object state, error
```

`stopping`ならGETを繰り返し、POSTを重ねません。`stopped`、errorなしに加えて、人が収納姿勢と異音・引っかかりがないことを確認します。物理電源の操作は別です。

## 出典

コマンドで呼び出したエンドポイントは、Pollen Roboticsの公式SDK [pollen-robotics/reachy_mini](https://github.com/pollen-robotics/reachy_mini) の[daemon](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/daemon.py)、[motors](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/motors.py)、[move](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/move.py)の各ルーターに由来します。実験時のSDK/daemonは1.13.0で、上記の観測結果は公式ソースの説明とは別の実機記録です。

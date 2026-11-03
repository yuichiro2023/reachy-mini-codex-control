# Reachy Mini Wireless — 起床・収納・停止

接続先：`http://reachy-mini.local:8000`  
使用環境：Windows PowerShell、公式SDK/daemon 1.13.0

環境の準備は[実行環境](docs/environment.md)、Codexとの進め方は[協働手順](docs/codex-workflow.md)を参照してください。

**各操作を個別に実行してください。一括実行用のスクリプトではありません。** Codexに実行させる場合は、GETによる状態確認も含め、コマンドと影響を確認して操作ごとに承認します。頭・アンテナの周囲から手や物を離してください。

## 1. daemonの状態を確認する

```powershell
$base = 'http://reachy-mini.local:8000'
$daemon = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
$daemon | Select-Object state, error, backend_status
```

- `stopped`：手順2でdaemonを起動します。
- `running`：起動操作を省き、手順3へ進みます。
- 応答なし：電源・Wi-Fi・ホスト名の名前解決を確認します。APIが応答しなければ、次の起動要求は使えません。

## 2. 停止中のdaemonを起動する

**事前条件：状態が`stopped`で、APIが応答していること。** `wake_up=false`を指定し、この段階では起床動作を行いません。

```powershell
$job = Invoke-RestMethod -Method Post -Uri "$base/api/daemon/start?wake_up=false" -TimeoutSec 10
$job.job_id
```

数秒後に確認します。`running`、backendの`ready=true`、daemon/backendのエラーなしが続行条件です。

```powershell
Start-Sleep -Seconds 3
$daemon = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
$daemon | Select-Object state, error, backend_status
```

## 3. 起床前の状態を確認する

```powershell
$app = Invoke-RestMethod -Method Get -Uri "$base/api/apps/current-app-status" -TimeoutSec 5
$lock = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/robot-app-lock-status" -TimeoutSec 5
$moves = Invoke-RestMethod -Method Get -Uri "$base/api/move/running" -TimeoutSec 5
$motors = Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
$startup = Invoke-RestMethod -Method Get -Uri "$base/api/apps/startup-app" -TimeoutSec 5
[pscustomobject]@{
    current_app = $app
    lock_state = $lock.state
    running_move_count = $moves.Count
    motor_mode = $motors.mode
    startup_app = $startup.startup_app
} | Format-List
```

続行条件は、管理アプリなし、ロック`free`、実行中動作0件です。自動起動アプリが設定されている場合は、起床後に動く可能性があるため、その内容を確認します。

## 4. モーターを有効にする

**事前条件：手順2・3の確認を終え、モーターが`disabled`であること。** モーターに力が入ります。実験者が周囲の安全を確認し、この操作を承認してから実行します。

```powershell
Invoke-RestMethod -Method Post -Uri "$base/api/motors/set_mode/enabled" -TimeoutSec 8
Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
```

`enabled`になったことと、急な動き・異音・引っかかりがないことを確認します。すでに`enabled`なら、このPOSTは省きます。

## 5. 起床する

**事前条件：モーターが`enabled`、daemon/backendが正常、管理アプリなし、ロック`free`、実行中動作0件。** 頭とアンテナが動きます。周囲の空間を再確認し、起床について別途承認を得てから1回だけ送ります。

```powershell
$task = Invoke-RestMethod -Method Post -Uri "$base/api/move/play/wake_up" -TimeoutSec 10
$task.uuid
```

```powershell
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$base/api/move/running" -TimeoutSec 5
Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
```

実験者が通常姿勢と、異音・引っかかりがないことを確認します。受付IDや動作0件という応答だけでは成功と判断できません。すでに通常姿勢なら、起床を繰り返しません。

## 6. 収納して停止する

**事前条件：使用中のアプリを終了し、手順1・3でdaemon/backend正常、管理アプリなし、ロック`free`、実行中動作0件を確認すること。**

頭とアンテナが収納姿勢へ動き、最後にモーターが無効になります。周囲を空け、収納・停止について承認を得てから実行します。

```powershell
$job = Invoke-RestMethod -Method Post -Uri "$base/api/daemon/stop?goto_sleep=true" -TimeoutSec 10
$job.job_id
```

```powershell
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5 |
    Select-Object state, error
```

`stopped`、エラーなしと、実機の収納姿勢を確認します。本体の電源スイッチは、この操作では切れません。

## 応答が遅い場合・異常がある場合

- `starting` / `stopping`の間はGETで経過を確認し、POSTを重ねません。
- タイムアウト時も受理済みの可能性があります。再送する前に状態を確認してください。
- モーター通信エラー、異音、引っかかり、過負荷、想定外の動きがあれば、追加の動作を指示しないでください。

## 出典と確認済みの範囲

2026-11-02 08:53〜08:56 JSTに、停止中からの起動、起床、収納・停止を実機で確認しました。SSHは使用していません。操作には公式SDK [pollen-robotics/reachy_mini](https://github.com/pollen-robotics/reachy_mini) のHTTP APIを使用しています。概要と成果の範囲は[README](README.md)を参照してください。

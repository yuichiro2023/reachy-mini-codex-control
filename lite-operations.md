# Reachy Mini Lite — 起床・収納・停止

接続先：`http://127.0.0.1:8000`（Liteを接続したWindows PC）  
使用環境：Windows PowerShell、公式SDK/daemon 1.13.0、USB接続

**各操作を個別に実行してください。一括実行用のスクリプトではありません。** Codexに実行させる場合は、GETによる状態確認も含め、コマンドと影響を確認して操作ごとに承認します。頭・アンテナの周囲から手や物を離してください。

## 1. daemonの状態を確認する

Liteの電源とUSB接続を確認してから実行します。

```powershell
$base = 'http://127.0.0.1:8000'
$daemon = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
$daemon | Select-Object state, error, backend_status
```

- `stopped`：手順2でdaemonの起動と起床を行います。
- `running`：起動操作を省きます。収納姿勢のままなら手順3へ進みます。
- 応答なし：末尾の「APIが応答しない場合」を参照してください。

## 2. 停止中から起動・起床する

**事前条件：状態が`stopped`で、APIが応答していること。** `wake_up=true`により、daemon起動、モーター有効化、起床を続けて行います。頭とアンテナが動くため、周囲の空間を確認し、この操作を承認してから実行します。

```powershell
Invoke-RestMethod -Method Post -Uri "$base/api/daemon/start?wake_up=true" -TimeoutSec 8
```

```powershell
Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
```

`running`、backendの`ready=true`、daemon/backendのエラーなし、モーター`enabled`を確認します。まだ起動中ならGETで経過を確認します。

実験者が通常姿勢と、異音・引っかかりがないことを確認します。**ここで起床できた場合、手順3のモーター有効化とwake-upは不要です。**

## 3. daemonが稼働中で、起床だけが必要な場合

### 状態を確認する

以下のコマンドは、収納・停止の前にも使います。

```powershell
$daemon = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
$motors = Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
$moves = Invoke-RestMethod -Method Get -Uri "$base/api/move/running" -TimeoutSec 5
$app = Invoke-RestMethod -Method Get -Uri "$base/api/apps/current-app-status" -TimeoutSec 5
$lock = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/robot-app-lock-status" -TimeoutSec 5
[pscustomobject]@{
    daemon_state = $daemon.state
    backend_ready = $daemon.backend_status.ready
    daemon_error = $daemon.error
    backend_error = $daemon.backend_status.error
    control_loop_errors = $daemon.backend_status.control_loop_stats.nb_error
    motor_mode = $motors.mode
    running_move_count = $moves.Count
    current_app = $app
    lock_state = $lock.state
} | Format-List
```

続行条件は、daemonが`running`、backendが準備完了、エラーなし、管理アプリなし、ロック`free`、実行中動作0件です。

### モーターを有効にする

**モーターが`disabled`の場合だけ実行します。** モーターに力が入るため、周囲の安全を確認し、この操作を承認してから送ります。

```powershell
Invoke-RestMethod -Method Post -Uri "$base/api/motors/set_mode/enabled" -TimeoutSec 8
Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
```

`enabled`になったことと、急な動き・異音・引っかかりがないことを確認します。

### 起床する

**事前条件：モーターが`enabled`で、上記の続行条件を満たしていること。** 頭とアンテナが動きます。周囲の空間を再確認し、起床について別途承認を得てから1回だけ送ります。

```powershell
$task = Invoke-RestMethod -Method Post -Uri "$base/api/move/play/wake_up" -TimeoutSec 10
$task | ConvertTo-Json -Compress
```

```powershell
Start-Sleep -Seconds 3
$daemon = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
$motors = Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
$moves = Invoke-RestMethod -Method Get -Uri "$base/api/move/running" -TimeoutSec 5
[pscustomobject]@{
    daemon_state = $daemon.state
    backend_ready = $daemon.backend_status.ready
    daemon_error = $daemon.error
    backend_error = $daemon.backend_status.error
    control_loop_errors = $daemon.backend_status.control_loop_stats.nb_error
    motor_mode = $motors.mode
    running_move_count = $moves.Count
} | Format-List
```

APIの正常応答に加え、実験者が通常姿勢と、異音・引っかかりがないことを確認します。受付IDや動作0件という応答だけでは成功と判断できません。

## 4. 収納して停止する

**事前条件：使用中のアプリを終了し、手順3の状態確認でdaemon/backend正常、管理アプリなし、ロック`free`、実行中動作0件を確認すること。**

頭とアンテナが収納姿勢へ動き、モーター制御とdaemonの内部処理が終了します。周囲を空け、収納・停止について承認を得てから実行します。

```powershell
Invoke-RestMethod -Method Post -Uri "$base/api/daemon/stop?goto_sleep=true" -TimeoutSec 8
```

```powershell
Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
```

`stopped`、エラーなしと、実機の収納姿勢を確認します。本体の電源スイッチは、この操作では切れません。

## APIが応答しない場合

WindowsでUSBシリアル機器の認識を確認します。実験時はCOM3でしたが、番号は環境によって変わります。

```powershell
Get-PnpDevice -Class Ports -PresentOnly |
    Where-Object FriendlyName -Match '\(COM\d+\)' |
    Select-Object Status, FriendlyName, InstanceId
```

COMポートが表示されても、daemonが通信できるとは限りません。過去にはCOMアクセス拒否とTCPポート8000の競合が発生しました。daemonを繰り返し起動せず、COMポートとTCPポート8000の使用状況を確認してください。APIが応答しない状態では、手順2の起動要求も使えません。

## 応答が遅い場合・異常がある場合

- `starting` / `stopping`の間はGETで経過を確認し、POSTを重ねません。
- タイムアウト時も受理済みの可能性があります。再送する前に状態を確認してください。
- モーター通信エラー、異音、引っかかり、過負荷、想定外の動きがあれば、追加の動作を指示しないでください。

## 出典と確認済みの範囲

2026-11-02 08:53〜08:56 JSTに、`start?wake_up=true`から`stop?goto_sleep=true`までの一連の操作を実機で確認しました。操作には公式SDK [pollen-robotics/reachy_mini](https://github.com/pollen-robotics/reachy_mini) のHTTP APIを使用しています。概要と成果の範囲は[README](README.md)を参照してください。

# Lite：公式HTTP APIによる起床・収納・daemon停止

Windows接続のReachy Mini Liteについて、2026-11-02の実測を公開用に整理しました。実験時のSDK/daemonは1.13.0です。管理APIは実行PCの`127.0.0.1:8000`で応答しました。記録された状態は再実行時の現在値ではありません。

## 実測結果と更新点

| 段階 | APIで観測したこと | 人による物理確認 |
|---|---|---|
| 先行する個別起床 | daemon `running`、backend ready、モーター`disabled`、動作0件、アプリなし、ロック`free`から開始。モーター有効化後`enabled`、wake-up POSTが動作UUIDを返し、その後の動作0件・errorなし・control-loop error 0 | 起床を確認 |
| 先行する収納・停止 | `stop?goto_sleep=true`の後、daemon `stopped`、error `null` | 寝姿勢への到達を確認 |
| 08:53の通常起動 | 停止中から`start?wake_up=true`の後、daemon `running`、backend ready、errorなし。後のGETでモーター`enabled`、control-loop error 0、メディア有効、ロック`free` | 通常姿勢、異音・引っかかりなし |
| 08:56の収納・停止 | `stop?goto_sleep=true`の後、daemon `stopped`、error `null` | 頭とアンテナが収納姿勢に到達、異音・引っかかりなし |

08:53–08:56 JSTの最新実測で、Liteの`start?wake_up=true`から`stop?goto_sleep=true`までの通常サイクルが確認されました。古い記録にある「通常サイクルは未試験」という判断は、この実測により更新されます。2026-11-01のLite音声会話は別の実験で、ここで扱う起床・収納には有料APIを使っていません。

USBシリアル機器は実験時に`COM3`としてWindowsに列挙されました。ただし、COM列挙だけではdaemonの通信成功は証明できません。以前のCodex環境からのdaemon CLI起動試行にはCOMアクセス拒否とTCP 8000競合があり、後からAPIが応答したプロセスの由来は未確認です。今回の成功は公式HTTP APIと実機観察で判定します。

## 再現時のコマンドと判定

GETもPOSTも実機への操作として、コマンドと影響を示した個別承認を得てから実行します。頭とアンテナの周囲を空け、異音、引っかかり、Motor communication error、過負荷、想定外の動きがあれば操作を重ねません。

### 1. Windows側とdaemonの事前状態を読む

Liteの電源・USB接続を確認します。COM番号は固定と決めず、その時点のWindowsの列挙を確認します。

```powershell
Get-PnpDevice -Class Ports -PresentOnly |
    Where-Object FriendlyName -Match '\(COM\d+\)' |
    Select-Object Status, FriendlyName, InstanceId
```

実機状態照会の承認後、daemonが応答するか確認します。

```powershell
$base = 'http://127.0.0.1:8000'
$daemon = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
$daemon | ConvertTo-Json -Depth 6
```

`running`ならdaemon開始POSTを重ねません。`stopped`でAPIが応答する場合のみ、通常起動の分岐へ進みます。APIに届かない場合、APIのstart要求は使えません。二重起動を避けるため、port 8000のlistenerとCOMの状態を調べ、原因が分かるまで起動を繰り返しません。

### 2. `stopped`からの通常起動とwake-up

事前GETで`stopped`を確認し、頭・アンテナの周囲から手や物を離します。`wake_up=true`はdaemonの起動、モーター有効化、起床の物理移動を伴います。その影響について個別承認を得た後のコマンドです。

```powershell
Invoke-RestMethod -Method Post -Uri "$base/api/daemon/start?wake_up=true" -TimeoutSec 8
Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
```

開始POSTは非同期ジョブの受付です。後のGETで`running`、backend `ready=true`、daemon/backend errorなしを確認し、さらにモーターの`enabled`を読みます。人が通常姿勢への到達、異音・引っかかりの有無を確認します。08:53 JSTの実測ではこの分岐に成功しました。POSTがタイムアウトしても、再送前に状態をGETします。

```powershell
Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
```

### 3. daemonが既に`running`で、起床だけ必要な場合

この分岐は先行するLiteの個別起床試験で使いました。まずdaemon/backend、モーター、動作、管理アプリ、ロック、メディアを読みます。状態照会の承認が必要です。

```powershell
$daemon = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
$motors = Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
$moves = Invoke-RestMethod -Method Get -Uri "$base/api/move/running" -TimeoutSec 5
$app = Invoke-RestMethod -Method Get -Uri "$base/api/apps/current-app-status" -TimeoutSec 5
$lock = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/robot-app-lock-status" -TimeoutSec 5
$media = Invoke-RestMethod -Method Get -Uri "$base/api/media/status" -TimeoutSec 5
[pscustomobject]@{
    daemon_state       = $daemon.state
    backend_ready      = $daemon.backend_status.ready
    daemon_error       = $daemon.error
    backend_error      = $daemon.backend_status.error
    motor_mode         = $motors.mode
    running_move_count = $moves.Count
    current_app        = $app
    lock_state         = $lock.state
    media_available    = $media.available
} | Format-List
```

続行条件はdaemon `running`、backend ready、errorなし、動作0件、アプリなし、ロック`free`です。モーターが`disabled`なら、トルク有効化の影響について別の個別承認を得て1回だけ送ります。

```powershell
Invoke-RestMethod -Method Post -Uri "$base/api/motors/set_mode/enabled" -TimeoutSec 8
Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
```

後のGETで`enabled`、daemon/backend正常、実機の異常なしを確認します。通常姿勢への移動がまだ必要なら、頭とアンテナが動くことを示し、起床について別の個別承認を得てから送ります。`start?wake_up=true`ですでに起きた場合は、このPOSTを追加しません。

```powershell
$task = Invoke-RestMethod -Method Post -Uri "$base/api/move/play/wake_up" -TimeoutSec 10
$task | ConvertTo-Json -Compress
```

受付UUIDだけでは成功と判定しません。数秒後に状態を読み、人が姿勢と異常の有無を確認します。

```powershell
Start-Sleep -Seconds 3
$daemon = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
$motors = Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
$moves = Invoke-RestMethod -Method Get -Uri "$base/api/move/running" -TimeoutSec 5
[pscustomobject]@{
    daemon_state        = $daemon.state
    backend_ready       = $daemon.backend_status.ready
    daemon_error        = $daemon.error
    backend_error       = $daemon.backend_status.error
    control_loop_errors = $daemon.backend_status.control_loop_stats.nb_error
    motor_mode          = $motors.mode
    running_move_count  = $moves.Count
} | Format-List
```

### 4. 収納姿勢へ動かしてdaemonを停止

使用中アプリを終了し、daemon/backend正常、実行中動作0件、アプリなし、ロック`free`を再GETします。頭とアンテナの周囲を空けます。`goto_sleep=true`は収納姿勢への物理移動、モーター制御とdaemon backendの終了を伴うため、影響について個別承認を得てから1回だけ送ります。

```powershell
Invoke-RestMethod -Method Post -Uri "$base/api/daemon/stop?goto_sleep=true" -TimeoutSec 8
Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
```

非同期ジョブなので、`stopping`ならGETで経過を確認します。`stopped`、errorなしと、人による収納姿勢・異常なしを確認します。タイムアウトや接続エラー時にも、状態を確認せず同じPOSTを再送しません。物理電源を切る操作は含みません。

## 出典

APIと起床・収納動作はPollen Roboticsの公式SDK [pollen-robotics/reachy_mini](https://github.com/pollen-robotics/reachy_mini) に由来します。参照先は[REST API資料](https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/API/rest-api.mdx)、[daemonルーター](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/daemon.py)、[daemon本体](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/daemon.py)です。ここに記したAPI応答と姿勢確認は、自作プログラムの結果ではなく実機実験の記録です。

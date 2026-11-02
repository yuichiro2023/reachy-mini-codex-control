# Liteを起こし、収納して停止する手順

この文書では、Windows PCに接続したReachy Mini Liteの操作を説明します。Codexは、PC上の管理APIにPowerShellから要求を送りました。実験者は、その場でロボットの姿勢や音を確認しました。実験時のSDK/daemonのバージョンは1.13.0です。

daemonはロボットを制御するプログラムです。APIの応答に出てくるbackendは、その内部で実機を制御する処理を指します。GETは状態の問い合わせ、POSTは起床や停止などの処理の依頼に使います。

## 実験で確認できたこと

2026年11月2日08:53（日本時間）、Codexは停止中のdaemonに対して、起床を伴う起動を指示しました。APIは稼働中を示し、内部の制御処理も準備を完了しました。実験者は、Liteが通常の姿勢になり、異音や引っかかりがないことを確認しました。

08:56には、Codexが収納と停止を指示しました。APIは停止済みを示し、実験者は頭とアンテナが収納姿勢になったことを確認しました。このときも、異音や引っかかりはありませんでした。

この実測によって、通常起動から収納・停止までの一連の操作を確認できました。以前の記録にあった「一連の操作は未試験」という記述は、今回の結果によって更新されます。

| 確認項目 | 起床後のAPIの応答 |
|---|---|
| daemonの状態 | running：制御プログラムが稼働しています。 |
| backendの状態 | ready=true：内部の制御処理が準備を完了しています。 |
| モーターの状態 | enabled：モーターが有効です。 |
| エラー | daemonとbackendにエラーはなく、制御ループのエラーも0件でした。 |
| その他 | メディアは有効で、ロックはfreeでした。管理アプリの応答はnullでした。 |

同日の先行試験では、すでに稼働しているdaemonに対し、モーターの有効化と起床を別々に指示して成功しました。後述の手順3は、このときの操作です。11月1日に行った音声会話は別の実験であり、今回の起床・収納には有料APIを使っていません。

## 操作前の確認

実験者は、Liteの電源とUSB接続を確認し、頭とアンテナの周囲から手や物を離します。Codexが実行する場合は、状態の問い合わせを含め、コマンドと影響を示して操作ごとの承認を得ます。

異音、引っかかり、モーター通信エラー、過負荷、想定外の動きが生じた場合は、追加の動作を指示しません。

### 1. PCとの接続とdaemonの状態を確認する

次のコマンドは、Windowsが認識しているシリアルポートを表示します。実験時には、実験者がCOM3として認識されていることを確認しました。ただし、番号は環境によって変わるため、再実行時には確認が必要です。

```powershell
Get-PnpDevice -Class Ports -PresentOnly |
    Where-Object FriendlyName -Match '\(COM\d+\)' |
    Select-Object Status, FriendlyName, InstanceId
```

続いて、Codexは管理APIにdaemonの状態を問い合わせます。

```powershell
$base = 'http://127.0.0.1:8000'
$daemon = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
$daemon | ConvertTo-Json -Depth 6
```

応答がstoppedなら、手順2に進みます。すでにrunningで、ロボットが収納姿勢のままなら、手順3で状態を確認します。APIから応答がない場合は、起動要求も送れないため、先に接続やポートの使用状況を調べます。

以前、Codexからdaemonをコマンドで起動しようとした際には、COMポートへのアクセス拒否や、TCPポート8000の競合が発生しました。その後、APIは応答するようになりましたが、どのプロセスがAPIを提供していたかは特定できていません。したがって、接続に失敗したときは起動を繰り返さず、COMポートとTCPポート8000を使用している処理を確認します。

### 2. 停止中のdaemonを起動し、Liteを起こす

Codexは、直前の問い合わせでdaemonがstoppedであることを確認します。実験者は、頭とアンテナの周囲が空いていることを確かめ、起床を伴う起動操作を承認します。

次のコマンドでは、wake_up=trueを指定します。これにより、daemonの起動に続いてモーターが有効になり、Liteが起き上がります。

```powershell
Invoke-RestMethod -Method Post -Uri "$base/api/daemon/start?wake_up=true" -TimeoutSec 8
Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
```

起動要求への応答は、処理の受付を示します。処理が終わるまで時間がかかる場合は、GETで状態を確認してください。Codexは、daemonがrunningになり、backendのreadyがtrueになり、両方にエラーがないことを確認します。

次のコマンドでは、モーターがenabledになったことを確認します。

```powershell
Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
```

実験者は、Liteが通常の姿勢になり、異音や引っかかりがないことを確認します。08:53の実験では、この方法で起床に成功しました。ここで起床できた場合は、手順3の操作を追加する必要はありません。

### 3. daemonが稼働中で、起床だけが必要な場合

Codexは、モーターを動かす前に、daemon、内部の制御処理、実行中の動作、アプリ、ロックの状態を問い合わせます。

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

次へ進む前に、daemonが稼働中で、backendの準備が完了し、エラーがないことを確認します。また、実行中の動作が0件、管理アプリがなし、ロックがfreeであることも確認します。ロックがfreeという応答は、管理上の操作権が占有されていないことを示します。

モーターがdisabledの場合は、Codexが実験者の承認を得て、次のコマンドで有効にします。この操作ではモーターに力が入るため、実験者は急な動きや異音がないか確認します。

```powershell
Invoke-RestMethod -Method Post -Uri "$base/api/motors/set_mode/enabled" -TimeoutSec 8
Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
```

Codexは、モーターがenabledになり、daemonとbackendにもエラーがないことを確認します。実験者が起床動作を別途承認したら、Codexは次のコマンドを1回だけ実行します。頭とアンテナが動くため、実験者は周囲の空間を再確認してください。

```powershell
$task = Invoke-RestMethod -Method Post -Uri "$base/api/move/play/wake_up" -TimeoutSec 10
$task | ConvertTo-Json -Compress
```

返されるUUIDは、動作要求を識別するための番号です。この番号だけでは、起床の成功は分かりません。数秒後、Codexは次のコマンドで状態を確認します。

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

実行中の動作が0件でも、動作が成功したとは限りません。実験者は、Liteが通常の姿勢になったことと、異音や引っかかりがないことを確認します。

### 4. Liteを収納してdaemonを停止する

使用中のアプリを終了したうえで、Codexは手順3の状態確認を行います。実行中の動作がなく、管理アプリもなく、ロックがfreeであり、daemonとbackendが正常であることを確認します。

実験者は頭とアンテナの周囲を空け、収納を伴う停止操作を承認します。次のコマンドは、Liteを収納姿勢へ動かしてから、モーター制御とdaemonの内部処理を終了します。

```powershell
Invoke-RestMethod -Method Post -Uri "$base/api/daemon/stop?goto_sleep=true" -TimeoutSec 8
Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
```

Codexは、状態がstoppedになり、エラーがないことを確認します。実験者は、頭とアンテナが収納姿勢になったことと、異常がないことを確認します。本体の電源スイッチを切る操作は、このコマンドには含まれません。

状態がstoppingの間は、GETで経過を確認します。タイムアウトや接続エラーが発生しても、処理がすでに受け付けられている可能性があるため、同じPOSTをすぐに再送しないでください。

## 出典

APIと起床・収納動作は、Pollen Roboticsの公式SDK [pollen-robotics/reachy_mini](https://github.com/pollen-robotics/reachy_mini) の機能です。詳細は[REST API資料](https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/API/rest-api.mdx)、[daemonのAPI実装](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/daemon.py)、[daemon本体の実装](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/daemon.py)を参照してください。リンク先は更新される場合があります。

この文書は、公式APIを使った操作と実機での確認結果を記録したものです。今回の実験のために、独自のプログラムは開発していません。

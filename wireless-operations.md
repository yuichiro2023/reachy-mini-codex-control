# Wirelessを起こし、収納して停止する手順

この文書では、Wi-Fiで接続したReachy Mini Wirelessの操作を説明します。Codexは、Windows PowerShellから本体の公式HTTP APIに要求を送りました。実験者は、その場でロボットの姿勢や音を確認しました。今回の操作にSSHは使用していません。

daemonはロボットを制御するプログラムです。APIの応答に出てくるbackendは、その内部で実機を制御する処理を指します。GETは状態の問い合わせ、POSTは起床や停止などの処理の依頼に使います。

## 実験で確認できたこと

2026年11月2日の先行試験では、daemonがすでに稼働している状態から、Codexがモーターの有効化と起床を指示しました。実験者は、Wirelessの頭とアンテナが通常の姿勢になり、異音や引っかかりがないことを確認しました。その後、収納とdaemon停止にも成功しました。

同日08:53（日本時間）には、停止中のdaemonを起動するところから試験しました。Codexは起床を伴わない起動を指示し、その後、モーターの有効化と起床を別々に行いました。実験者は、Wirelessが通常の姿勢になり、異常がないことを確認しました。

08:56には、Codexが収納と停止を指示しました。APIは停止済みを示し、実験者は収納姿勢と異常がないことを確認しました。2体を操作した順序は、[2体の実験記録](pair-start-stop.md)にまとめています。

| 段階 | APIで確認した結果 |
|---|---|
| 先行試験の起床前 | daemonはrunning、backendはready=trueでした。エラーはありませんでした。管理アプリと実行中の動作はなく、ロックはfree、モーターはdisabledでした。自動起動アプリはnullでした。 |
| モーターの有効化 | POSTの応答はHTTP 200で、その後の問い合わせではモーターがenabledになっていました。 |
| 起床 | POSTの応答はHTTP 200で、動作の受付IDが返りました。その後、実行中の動作は0件になりました。 |
| 08:53の再起動 | daemonはrunning、backendは準備完了となり、エラーはありませんでした。モーターの有効化と起床後には、実行中の動作が0件になりました。 |
| 08:56の停止 | stateはstopped、errorはnullでした。 |

APIが要求を受け付けたことや、実行中の動作が0件になったことだけでは、目的の姿勢になったかは分かりません。この実験では、実験者の観察と合わせて成功を判断しました。停止したのはdaemonであり、本体の電源スイッチは操作していません。

## 操作前の確認

実験者は、Wirelessの電源とWi-Fi接続を確認し、頭とアンテナの周囲から手や物を離します。Codexが実行する場合は、状態の問い合わせを含め、コマンドと影響を示して操作ごとの承認を得ます。

以下のコマンドを一括で実行せず、各段階でAPIの応答と実機の状態を確認してください。

### 1. daemonの状態を確認する

Codexは、次のコマンドでWirelessの管理APIに状態を問い合わせます。変数$baseには、本体のAPIの接続先を指定しています。

```powershell
$base = 'http://reachy-mini.local:8000'
$daemon = Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5
$daemon | Select-Object state, error, camera_specs_name, backend_status
```

APIから応答がない場合は、API経由でdaemonを起動できません。ホスト名から本体に接続できるか、Wi-Fiがつながっているか、電源が入っているかを確認してください。すでにdaemonがrunningなら、手順2の起動操作は省きます。

### 2. 停止中のdaemonを起動する

Codexは、状態がstoppedであることを確認し、実験者の承認を得て次のコマンドを実行します。wake_up=falseを指定するため、この段階ではdaemonだけが起動し、ロボットは起き上がりません。

```powershell
$job = Invoke-RestMethod -Method Post -Uri "$base/api/daemon/start?wake_up=false" -TimeoutSec 10
$job.job_id
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5 |
    Select-Object state, error, backend_status
```

返されるjob_idは、要求の受付番号です。Codexは、その後の問い合わせでdaemonがrunningになり、backendの準備が完了し、エラーがないことを確認します。まだstartingなら、GETで経過を確認してください。起動のPOSTを繰り返す必要はありません。

08:53の実験では、この方法で停止中のdaemonを起動できました。

### 3. ほかの処理が動いていないか確認する

Codexは、daemonとbackendが正常であることを確かめた後、実行中のアプリや動作、モーターなどの状態を問い合わせます。

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

次へ進む前に、管理アプリがなく、実行中の動作が0件で、ロックがfreeであることを確認します。ロックがfreeという応答は、管理上の操作権が占有されていないことを示します。ただし、APIで確認できない外部プログラムの接続まで否定できるわけではありません。

自動起動アプリが設定されている場合は、起床後にそのアプリが動く可能性があります。実行前に、設定内容と影響を確認してください。

### 4. モーターを有効にしてから起床を指示する

モーターがdisabledの場合は、Codexが実験者の承認を得て、次のコマンドで有効にします。モーターに力が入るため、実験者は急な動きや異音がないか確認します。

```powershell
Invoke-RestMethod -Method Post -Uri "$base/api/motors/set_mode/enabled" -TimeoutSec 8
Invoke-RestMethod -Method Get -Uri "$base/api/motors/status" -TimeoutSec 5
```

Codexは、モーターがenabledになったことを確認します。実験者が起床動作を別途承認したら、Codexは次のコマンドを1回だけ実行します。

実行前には、動作やアプリが実行中でなく、ロックがfreeであることを再確認します。また、頭とアンテナが動くため、実験者は周囲の空間を確かめてください。

```powershell
$task = Invoke-RestMethod -Method Post -Uri "$base/api/move/play/wake_up" -TimeoutSec 10
$task.uuid
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$base/api/move/running" -TimeoutSec 5
```

CodexはAPIの応答を確認し、実験者はWirelessが通常の姿勢になったことと、異音や引っかかりがないことを確認します。すでに通常の姿勢でモーターも有効なら、起床操作を繰り返しません。

タイムアウトやHTTPエラーが出た場合は、処理がすでに受け付けられている可能性があります。同じPOSTをすぐに再送せず、状態を問い合わせてください。異常な動きがあった場合も、追加の動作を指示しないでください。

### 5. Wirelessを収納してdaemonを停止する

Codexは、使用中のアプリや実行中の動作がなく、ロックがfreeであり、daemonとbackendにエラーがないことを再確認します。実験者は、頭とアンテナの周囲を空け、収納を伴う停止操作を承認します。

次のコマンドは、Wirelessを収納姿勢へ動かしてからdaemonを停止します。収納動作の最後にはモーターが無効になります。

```powershell
$job = Invoke-RestMethod -Method Post -Uri "$base/api/daemon/stop?goto_sleep=true" -TimeoutSec 10
$job.job_id
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$base/api/daemon/status" -TimeoutSec 5 |
    Select-Object state, error
```

Codexは、状態がstoppedになり、エラーがないことを確認します。実験者は、頭とアンテナが収納姿勢になり、異音や引っかかりがないことを確認します。

状態がstoppingの間は、GETで経過を確認し、停止のPOSTを繰り返さないでください。本体の電源スイッチを切る操作は、このコマンドには含まれません。

## 出典

APIは、Pollen Roboticsの公式SDK [pollen-robotics/reachy_mini](https://github.com/pollen-robotics/reachy_mini) の機能です。参照先は、[daemonのAPI実装](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/daemon.py)、[モーターのAPI実装](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/motors.py)、[動作のAPI実装](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/move.py)です。

実験時のSDK/daemonは1.13.0です。リンク先は更新される場合があります。この文書に記載したAPIの応答と実験者の観察は、2026年11月2日の実機記録です。

# 2体のReachy Miniを起こし、収納して停止するまで

2026年11月2日08:53〜08:56（日本時間）、CodexはWindows PowerShellから公式HTTP APIを呼び出し、WirelessとLiteを順番に操作しました。実験者は、各操作を承認したうえで、ロボットの姿勢と音をその場で確認しました。

この実験では、Wireless、Liteの順に起床させました。両機が通常の姿勢になったことを確認した後、Lite、Wirelessの順に収納し、daemonを停止しました。daemonとは、ロボットを制御するプログラムのことです。

## 起床時の結果（08:53）

Wirelessでは、Codexが停止中のdaemonを起動しました。このときはwake_up=falseを指定し、起床動作を後の操作に分けました。続いて、Codexはモーターを有効にし、wake-upを指示しました。実験者は、Wirelessが通常の姿勢になり、異音や引っかかりがないことを確認しました。

Liteでは、Codexがwake_up=trueを指定してdaemonを起動しました。この指定によって、daemonの起動に続いて起床動作が実行されました。実験者は、Liteも通常の姿勢になり、異音や引っかかりがないことを確認しました。

| APIで確認した項目 | Wireless | Lite |
|---|---|---|
| 制御プログラムの状態 | running（稼働中） | running（稼働中） |
| 内部の制御処理の準備 | backend ready（準備完了） | backend ready（準備完了） |
| エラーの有無 | エラーなし | エラーなし |
| モーターの状態 | enabled（有効） | enabled（有効） |
| その他の確認 | wake-upの受付IDを受け取り、その後の実行中動作は0件でした。 | 制御ループのエラーは0件でした。メディアは有効、ロックはfree、管理アプリはnullでした。 |

Liteについては、Codexの実行環境でアクセスが拒否されたため、CIMによるCOMポート一覧の取得はできませんでした。それでも、公式HTTP APIによるdaemonの起動と、実機の起床は成功しました。

この結果により、Liteの通常起動から収納・停止までの一連の操作を、実機で確認できました。以前の記録にある「一連の操作は未試験」という記述は、この結果によって更新されます。

## 収納・停止時の結果（08:56）

Codexは、まずLiteに収納と停止を指示しました。実験者がLiteの収納姿勢を確認した後、CodexはWirelessにも同じ操作を行いました。

| 対象 | APIの応答 | 実験者が確認したこと |
|---|---|---|
| Lite | stateはstopped（停止済み）、errorはnull（エラーなし）でした。 | 頭とアンテナが収納姿勢になりました。異音や引っかかりはありませんでした。 |
| Wireless | stateはstopped（停止済み）、errorはnull（エラーなし）でした。 | 頭とアンテナが収納姿勢になりました。異音や引っかかりはありませんでした。 |

この操作で停止したのはdaemonです。実験者は、本体の電源スイッチを操作していません。

## 使用したコマンドと操作の順序

以下では、実際に使ったHTTP APIを、2体の接続先が混ざらないように整理しています。コマンドは一括で実行せず、各段階で結果を確認してから次に進みます。Codexが実行する場合は、状態の問い合わせも含め、実験者から操作ごとの承認を得ます。

### 1. Wirelessの状態を確認する

実験者は、Wirelessの電源とWi-Fi接続を確認し、頭とアンテナの周囲を空けます。Codexは、次のGET要求でdaemonの状態を問い合わせます。

```powershell
$wireless = 'http://reachy-mini.local:8000'
Invoke-RestMethod -Method Get -Uri "$wireless/api/daemon/status" -TimeoutSec 5
```

応答がstoppedなら、Codexは実験者の承認を得て、次のコマンドでdaemonを起動します。すでにrunningなら、この起動操作は省きます。APIから応答がない場合は、API経由で起動できないため、接続状態を確認します。

```powershell
Invoke-RestMethod -Method Post -Uri "$wireless/api/daemon/start?wake_up=false" -TimeoutSec 10
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$wireless/api/daemon/status" -TimeoutSec 5
```

### 2. Wirelessを起こす

Codexは、daemonが稼働中で、内部の制御処理の準備が完了し、エラーがないことを確認します。さらに、ほかのアプリや動作が実行中でないこと、ロックがfreeであること、起床時の自動起動アプリの設定を調べます。確認用のコマンドは[Wirelessの操作手順](wireless-operations.md)に記載しています。

モーターがdisabledの場合は、Codexが実験者の承認を得てモーターを有効にします。この操作によってモーターに力が入るため、実験者は急な動きや異音がないか確認します。

```powershell
Invoke-RestMethod -Method Post -Uri "$wireless/api/motors/set_mode/enabled" -TimeoutSec 8
Invoke-RestMethod -Method Get -Uri "$wireless/api/motors/status" -TimeoutSec 5
```

Codexは、モーターがenabledになったことを確認します。実験者が起床動作を別途承認したら、Codexは次のコマンドを1回だけ実行します。頭とアンテナが動くため、実験者は周囲の空間を再確認してください。

```powershell
Invoke-RestMethod -Method Post -Uri "$wireless/api/move/play/wake_up" -TimeoutSec 10
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$wireless/api/move/running" -TimeoutSec 5
```

実験者は、Wirelessが通常の姿勢になったことと、異音や引っかかりがないことを確認します。すでに通常の姿勢でモーターも有効な場合は、起床操作を繰り返しません。

### 3. Liteを起こす

実験者は、Liteの電源とUSB接続を確認し、頭とアンテナの周囲を空けます。Codexは、Windows PC上の管理APIに状態を問い合わせます。

```powershell
$lite = 'http://127.0.0.1:8000'
Invoke-RestMethod -Method Get -Uri "$lite/api/daemon/status" -TimeoutSec 5
```

応答がstoppedの場合は、Codexが実験者の承認を得て次のコマンドを実行します。この操作では、daemonの起動に続いて、ロボットが起き上がります。すでにrunningなら、起動コマンドを繰り返しません。

```powershell
Invoke-RestMethod -Method Post -Uri "$lite/api/daemon/start?wake_up=true" -TimeoutSec 8
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$lite/api/daemon/status" -TimeoutSec 5
Invoke-RestMethod -Method Get -Uri "$lite/api/motors/status" -TimeoutSec 5
```

Codexは、daemonがrunning、内部の制御処理が準備完了、モーターがenabledであり、エラーがないことを確認します。実験者は、Liteが通常の姿勢になったことと、異常がないことを確認します。

APIから応答がない場合は、[Liteの操作手順](lite-operations.md)に従って接続やポートの使用状況を調べます。

### 4. Liteを収納して停止する

Codexは、実行中のアプリや動作がないこと、ロックがfreeであること、daemonと内部の制御処理にエラーがないことを再確認します。実験者は、頭とアンテナの周囲を空け、収納を伴う停止操作を承認します。

```powershell
Invoke-RestMethod -Method Post -Uri "$lite/api/daemon/stop?goto_sleep=true" -TimeoutSec 8
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$lite/api/daemon/status" -TimeoutSec 5
```

Codexは、状態がstoppedになり、エラーがないことを確認します。実験者は、Liteの収納姿勢と、異音や引っかかりがないことを確認します。

### 5. Wirelessを収納して停止する

Liteの停止を確認した後、CodexはWirelessにも同じ事前確認を行います。実験者がWirelessの収納・停止を別途承認したら、次のコマンドを実行します。

```powershell
Invoke-RestMethod -Method Post -Uri "$wireless/api/daemon/stop?goto_sleep=true" -TimeoutSec 10
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$wireless/api/daemon/status" -TimeoutSec 5
```

Codexは、状態がstoppedになり、エラーがないことを確認します。実験者は、Wirelessの収納姿勢と、異音や引っかかりがないことを確認します。

状態がstoppingの間は、停止処理が進んでいるため、GETで経過を確認します。タイムアウトした場合も、処理が受け付けられている可能性があります。同じPOSTをすぐに再送せず、状態を調べてください。モーター通信エラーや異音、過負荷、想定外の動きが発生した場合は、追加の動作を指示しないでください。

## この実験で確認できた範囲

今回、2体の起床・収納・daemon停止を実機で確認できました。2体が撮影や会話を連携して行うことは、この実験では確認していません。

操作にはPollen Roboticsの公式SDK [pollen-robotics/reachy_mini](https://github.com/pollen-robotics/reachy_mini) のHTTP APIを使用しました。Wirelessの起床にSSHは使用していません。今回の成果は、公式APIを使った手順と実測記録です。

# 2体を順番に起こし、収納して停止する

**起床：Wireless → Lite**  
**収納・停止：Lite → Wireless**

Windows PowerShellから、各機の公式HTTP APIを呼び出します。以下は操作順序のガイドです。各ブロックを個別に実行し、確認を挟んでください。一括実行用のスクリプトではありません。

Codexに実行させる場合は、GETによる状態確認も含め、コマンドと影響を示して操作ごとに承認を得ます。頭とアンテナが動く操作では、実験者が周囲の空間を確保してください。

## 接続先を設定する

```powershell
$wireless = 'http://reachy-mini.local:8000'
$lite = 'http://127.0.0.1:8000'
```

Wirelessは本体のAPI、LiteはWindows PC上のAPIです。接続構成は[README](README.md)を参照してください。リンク先の詳細手順にある状態確認コマンドを使うときは、Wirelessでは`$base = $wireless`、Liteでは`$base = $lite`を設定します。

## 1. Wirelessのdaemonを起動する

まず、状態を確認します。

```powershell
Invoke-RestMethod -Method Get -Uri "$wireless/api/daemon/status" -TimeoutSec 5
```

**状態が`stopped`で、APIが応答している場合だけ**、起動について承認を得て次を実行します。`wake_up=false`のため、この段階では起床しません。`running`なら起動を省きます。

```powershell
Invoke-RestMethod -Method Post -Uri "$wireless/api/daemon/start?wake_up=false" -TimeoutSec 10
```

```powershell
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$wireless/api/daemon/status" -TimeoutSec 5
```

`running`、backendの準備完了、エラーなしを確認します。

## 2. Wirelessを起こす

[Wirelessの状態確認コマンド](wireless-operations.md#3-起床前の状態を確認する)で、管理アプリなし、ロック`free`、実行中動作0件、モーター状態、自動起動アプリの設定を確認します。

**モーターが`disabled`の場合だけ**、モーター有効化の承認を得て実行します。モーターに力が入るため、急な動きや異音に注意してください。

```powershell
Invoke-RestMethod -Method Post -Uri "$wireless/api/motors/set_mode/enabled" -TimeoutSec 8
Invoke-RestMethod -Method Get -Uri "$wireless/api/motors/status" -TimeoutSec 5
```

`enabled`と実機の異常なしを確認してから、起床について別途承認を得ます。daemon/backendが正常で、アプリや動作が実行中でないことを再確認してください。頭とアンテナが動きます。

```powershell
Invoke-RestMethod -Method Post -Uri "$wireless/api/move/play/wake_up" -TimeoutSec 10
```

```powershell
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$wireless/api/move/running" -TimeoutSec 5
Invoke-RestMethod -Method Get -Uri "$wireless/api/daemon/status" -TimeoutSec 5
```

実験者が通常姿勢と、異音・引っかかりがないことを確認します。すでに通常姿勢なら、起床を繰り返しません。

## 3. Liteを起こす

Liteの電源とUSB接続を確認してから、状態を問い合わせます。

```powershell
Invoke-RestMethod -Method Get -Uri "$lite/api/daemon/status" -TimeoutSec 5
```

**状態が`stopped`で、APIが応答している場合だけ**、起床を伴う起動について承認を得て実行します。頭とアンテナが動くため、周囲を空けてください。

```powershell
Invoke-RestMethod -Method Post -Uri "$lite/api/daemon/start?wake_up=true" -TimeoutSec 8
```

```powershell
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$lite/api/daemon/status" -TimeoutSec 5
Invoke-RestMethod -Method Get -Uri "$lite/api/motors/status" -TimeoutSec 5
```

`running`、backendの準備完了、エラーなし、モーター`enabled`を確認します。実験者が通常姿勢と、異音・引っかかりがないことを確認します。

すでに`running`で収納姿勢のままなら、[Liteの操作手順](lite-operations.md)の「3. daemonが稼働中で、起床だけが必要な場合」を使います。起動要求を繰り返さないでください。

## 4. Liteを収納して停止する

**事前条件：** 使用中のアプリを終了し、[Liteの状態確認コマンド](lite-operations.md#状態を確認する)で、daemon/backend正常、管理アプリなし、ロック`free`、実行中動作0件を確認します。

頭とアンテナが収納姿勢へ動きます。周囲を空け、Liteの収納・停止について承認を得て実行します。

```powershell
Invoke-RestMethod -Method Post -Uri "$lite/api/daemon/stop?goto_sleep=true" -TimeoutSec 8
```

```powershell
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$lite/api/daemon/status" -TimeoutSec 5
```

`stopped`、エラーなしと、実機の収納姿勢・異常なしを確認してから、Wirelessの停止へ進みます。

## 5. Wirelessを収納して停止する

**事前条件：** 使用中のアプリを終了し、[Wirelessの手順1・3](wireless-operations.md)で、daemon/backend正常、管理アプリなし、ロック`free`、実行中動作0件を確認します。

頭とアンテナが収納姿勢へ動きます。周囲を空け、Wirelessの収納・停止について別途承認を得て実行します。

```powershell
Invoke-RestMethod -Method Post -Uri "$wireless/api/daemon/stop?goto_sleep=true" -TimeoutSec 10
```

```powershell
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$wireless/api/daemon/status" -TimeoutSec 5
```

`stopped`、エラーなしと、実機の収納姿勢・異常なしを確認します。両機とも、この操作で本体の電源スイッチは切れません。

## 共通の注意事項

- APIが応答しない場合は、接続状態を確認します。LiteではUSB接続とポートの使用状況も確認してください。
- `starting` / `stopping`やタイムアウト時は、GETで状態を確認し、POSTをすぐに再送しないでください。
- 受付IDや実行中動作0件という応答だけでは、目的の姿勢になったとは判断できません。実機を目視で確認してください。
- モーター通信エラー、異音、引っかかり、過負荷、想定外の動きがあれば、追加の動作を指示しないでください。

2026-11-02 08:53 JSTに両機の起床、08:56 JSTにLite→Wirelessの収納・停止を実機で確認しました。APIの正常応答に加え、実験者が各姿勢と異常なしを確認しています。使用したAPIは公式SDK [pollen-robotics/reachy_mini](https://github.com/pollen-robotics/reachy_mini) の機能です。

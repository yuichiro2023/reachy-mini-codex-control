# 2体を順番に起床・収納・停止した実測

2026-11-02 08:53–08:56 JST、Windows PowerShellからReachy Mini WirelessとLiteの公式HTTP管理APIを使った実験です。起床はWireless→Lite、収納・daemon停止はLite→Wirelessの順でした。各段階でコマンドと影響を示し、個別承認を得て実行しました。以下の状態は当時の観測で、再実行時の現在値ではありません。

## 08:53 JST：起床

| 順序 | API上の実測 | 人による物理確認 |
|---|---|---|
| 1. Wireless | 停止中から`POST /api/daemon/start?wake_up=false`。後のGETでdaemon `running`、backend ready、errorなし。モーター有効化後`enabled`。wake-up要求は動作UUIDを返し、後の実行中動作0件 | 通常姿勢に到達、異音・引っかかりなし |
| 2. Lite | ローカルAPIで`stopped`を確認してから`POST /api/daemon/start?wake_up=true`。後のGETでdaemon `running`、backend ready、errorなし。モーター`enabled`、control-loop error 0、メディア有効、ロック`free`、管理アプリのAPI本文はJSON `null` | 通常姿勢に到達、異音・引っかかりなし |

LiteのCOMポートはCodex環境のCIMアクセス拒否で列挙できませんでしたが、公式daemon APIでの起動と物理的な起床は確認できました。この最新結果により、以前の「Liteの通常起動から停止までの連続実機試験は未実施」という判断は更新されます。

## 08:56 JST：収納・daemon停止

| 順序 | API上の実測 | 人による物理確認 |
|---|---|---|
| 1. Lite | `POST /api/daemon/stop?goto_sleep=true`。後のGETで`state=stopped`、error `null` | 頭・アンテナが収納姿勢に到達、異音・引っかかりなし |
| 2. Wireless | `POST /api/daemon/stop?goto_sleep=true`。後のGETで`state=stopped`、error `null` | 頭・アンテナが収納姿勢に到達、異音・引っかかりなし |

両方ともdaemon内部を停止しました。物理的な電源スイッチは操作していません。非同期ジョブの受付と実行中動作0件だけでは姿勢への到達を証明できないため、人の観察を別に記録しました。

## 実行順序と実際に使ったAPIコマンド

以下は実際に使ったHTTPエンドポイントを、2体の接続先が混ざらないようにPowerShellで整理したものです。各GETも実機の状態照会なので個別承認を得ます。POSTはまとめて実行せず、各段階の状態と人による物理確認を挟みます。接続先はWireless本体とLiteを接続したPCで異なります。実験で記録された元のPowerShell例は各機の手順にも残しています。

### 1. Wirelessを起こす

人が本体の電源とWi-Fi、頭・アンテナの周囲の空間を確認します。まずdaemonをGETし、`stopped`なら開始POST、`running`なら開始POSTを省きます。HTTPに届かなければAPIで起動できません。`wake_up=false`はこの段階では物理的な起床を行いません。

```powershell
$wireless = 'http://reachy-mini.local:8000'
Invoke-RestMethod -Method Get -Uri "$wireless/api/daemon/status" -TimeoutSec 5
```

`stopped`と確認した場合にのみ、daemon起動について承認を得て次を実行します。

```powershell
Invoke-RestMethod -Method Post -Uri "$wireless/api/daemon/start?wake_up=false" -TimeoutSec 10
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$wireless/api/daemon/status" -TimeoutSec 5
```

後のGETで`running`、backend ready、errorなしを確認してから、アプリなし、ロック`free`、実行中動作0件、モーター状態と自動起動アプリを読みます。具体的なGETは[Wireless手順](wireless-operations.md)に記載しています。モーターが`disabled`ならトルク有効化について承認を得てPOSTし、GETで`enabled`と異常なしを確認します。

```powershell
Invoke-RestMethod -Method Post -Uri "$wireless/api/motors/set_mode/enabled" -TimeoutSec 8
Invoke-RestMethod -Method Get -Uri "$wireless/api/motors/status" -TimeoutSec 5
```

頭・アンテナの動きを伴うwake-upについて別の承認を得て、周囲の空間と競合状態を確認してから1回だけ送ります。

```powershell
Invoke-RestMethod -Method Post -Uri "$wireless/api/move/play/wake_up" -TimeoutSec 10
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$wireless/api/move/running" -TimeoutSec 5
```

人が通常姿勢への到達、異音・引っかかりがないことを確認します。既に`enabled`で通常姿勢なら、これらのPOSTを重ねません。

### 2. Liteを起こす

人がLiteの電源・USB接続、頭・アンテナの空間を確認します。Liteの管理APIはPC上のlocalhostです。事前GETで`stopped`かつAPI応答ありと確認した場合に限り、daemon起動と物理的な起床を伴う`wake_up=true`の影響について個別承認を得てPOSTします。

```powershell
$lite = 'http://127.0.0.1:8000'
Invoke-RestMethod -Method Get -Uri "$lite/api/daemon/status" -TimeoutSec 5
```

`stopped`を確認した場合にのみ、起床を伴うdaemon開始について承認を得て次を実行します。

```powershell
Invoke-RestMethod -Method Post -Uri "$lite/api/daemon/start?wake_up=true" -TimeoutSec 8
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$lite/api/daemon/status" -TimeoutSec 5
Invoke-RestMethod -Method Get -Uri "$lite/api/motors/status" -TimeoutSec 5
```

`running`ならstartを重ねません。後のGETで`running`、backend ready、errorなし、モーター`enabled`を確認し、人が通常姿勢と異常の有無を確認します。APIが応答しなければ、[Lite手順](lite-operations.md)のCOM・ポート競合の調査に戻ります。

### 3. Lite、次にWirelessを収納・停止

各機で使用中アプリ、実行中動作、ロック、daemon/backend状態を再確認し、周囲を空けます。停止POSTは頭・アンテナの収納移動とdaemon停止を伴います。Liteの停止について個別承認を得て実行・確認し、人が収納姿勢と異常なしを確認してから、Wirelessの停止について別の個別承認を得ます。

```powershell
Invoke-RestMethod -Method Post -Uri "$lite/api/daemon/stop?goto_sleep=true" -TimeoutSec 8
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$lite/api/daemon/status" -TimeoutSec 5
```

```powershell
Invoke-RestMethod -Method Post -Uri "$wireless/api/daemon/stop?goto_sleep=true" -TimeoutSec 10
Start-Sleep -Seconds 3
Invoke-RestMethod -Method Get -Uri "$wireless/api/daemon/status" -TimeoutSec 5
```

どちらも`stopped`、errorなしと物理的な収納姿勢を確認します。`stopping`やタイムアウト時はGETで状態を確認し、POSTを盲目的に再送しません。Motor communication error、異音、引っかかり、過負荷、想定外の動きがあれば次の移動を止めます。

## 範囲と出典

この実験では2体の起床・収納・daemon停止を確認しました。2体が撮影や会話で連携する一往復は、この記録からは実証できません。今回のWireless wake-upは公式HTTP APIで成功しており、SSHは必須手順ではありません。APIとdaemonの実装はPollen Roboticsの公式SDK [pollen-robotics/reachy_mini](https://github.com/pollen-robotics/reachy_mini) に属し、この文書に示したPowerShellの操作記録は自作プログラムではありません。

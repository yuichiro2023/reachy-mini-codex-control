# Physical AI最終課題：2体のReachy Miniの起床・収納・停止

WindowsのCodex CLIから、Reachy Mini WirelessとReachy Mini Liteを順番に起こし、収納姿勢へ戻してdaemonを停止した実機実験の公開記録です。2026-11-02 08:53–08:56 JSTの観測を基準に整理しました。ここに記した状態は実験時点のもので、現在の実機状態を示しません。

## 何を行い、何を確認したか

1. Wirelessのdaemonを公式HTTP APIで起動し、状態を確認した後、モーターを有効化してwake-upを実行しました。
2. Liteのdaemonを公式HTTP APIの`wake_up=true`で起動しました。
3. 人が両機の通常姿勢への到達と、異音・引っかかりがないことを確認しました。
4. Lite、Wirelessの順に公式HTTP APIの`goto_sleep=true`で収納・daemon停止を実行しました。人が両機の収納姿勢と異常がないことを確認しました。

両機のdaemonは停止状態になりましたが、物理的な電源スイッチは操作していません。HTTP要求の受付や実行中動作0件という表示だけで物理的な成功を判断せず、人の観察と組み合わせています。2体が会話や撮影を連携して一往復する実験は、この公開記録の対象外です。

## 公開ファイル

| ファイル | 内容 |
|---|---|
| [pair-start-stop.md](pair-start-stop.md) | 2体を扱った順序、08:53–08:56 JSTの実測、共通の安全確認 |
| [wireless-operations.md](wireless-operations.md) | WirelessのHTTP APIコマンド、事前確認、実測と判定範囲 |
| [lite-operations.md](lite-operations.md) | LiteのHTTP APIコマンド、事前確認、実測と判定範囲 |

## 成果物と出典の境界

この起床・収納・停止実験のための独自プログラムは開発していません。成果物は、Codex CLIで調べて提示・実行したPowerShellコマンド、公式APIを使う操作順序、API応答と人による物理確認を区別した記録です。別目的のコードがプロジェクト内に存在しても、今回の操作を自作コードの実行成果とは扱いません。

APIとdaemonの実装はPollen Roboticsの公式SDK [pollen-robotics/reachy_mini](https://github.com/pollen-robotics/reachy_mini) に属します。実験時のSDK/daemonは1.13.0です。参照先として公式の[REST API資料](https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/API/rest-api.mdx)、[daemonルーター](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/daemon.py)、[モータールーター](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/motors.py)、[動作ルーター](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/app/routers/move.py)を示します。リンク先の`main`は閲覧時に更新され得るため、本書の実測値は2026-11-02時点の記録として読んでください。

今回のWirelessの成功経路は公式HTTP APIで、SSHを使用していません。両機の起床・収納・daemon停止にOpenAI APIキーも不要でした。

## 再実行時の安全条件

コマンドはWindows PowerShell用です。Wirelessは`http://reachy-mini.local:8000`、Liteは実行PC上の`http://127.0.0.1:8000`を使いました。どちらも実行前にその時点の接続先と状態を確認してください。GETを含む実機への接続・状態照会、daemon操作、モーター操作、移動は、それぞれ影響を示して個別承認を得る運用でした。頭とアンテナの周囲を空け、異音・引っかかり・過負荷・想定外の動きがあれば追加の移動やPOSTの再送を止めて状態を確認します。

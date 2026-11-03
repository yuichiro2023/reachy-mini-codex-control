# Physical AI最終課題：Codex CLIで2体のReachy Miniを操作

本課題では、Codex CLIを使って2体の[Reachy Mini](https://pollen-robotics.com/reachy-mini/)を操作しました。対象は、Windows PCにUSB接続した[Reachy Mini Lite](https://store.pollen-robotics.com/products/reachy-mini-lite-version)と、Wi-Fi経由で接続した[Reachy Mini Wireless](https://store.pollen-robotics.com/products/reachy-mini-wireless-version)です（以下、それぞれ「Lite」「Wireless」と表記）。Codexは公式HTTP APIにコマンドを送り、実験者はロボットの姿勢や音をその場で確認しました。

2026年11月2日、両機は通常姿勢への起動動作を行いました。その後、頭とアンテナを収納姿勢に戻し、制御プログラムも停止しました。

## Reachy Miniとは

Reachy MiniはPollen Roboticsの小型オープンソースロボットです。頭部・胴体・アンテナを動かせるほか、カメラ、マイク、スピーカーを備え、Python SDKなどから開発できます。今回の検証対象は、そのLiteとWirelessです。製品の詳細は[公式SDK](https://github.com/pollen-robotics/reachy_mini)を参照してください。

| | Lite | Wireless |
|---|---|---|
| 製品としての違い | 外部コンピューターにUSB接続して利用 | 本体内のコンピューターを利用し、Wi-Fiで接続可能 |
| 今回のdaemon実行場所 | Windows PC | Wireless本体のLinux |
| 今回の操作経路 | PC内のHTTP APIからUSB経由 | PCからWi-Fi/LAN経由で本体のHTTP APIへ |

## 接続構成

```mermaid
flowchart TB
    user["実験者：自然言語で指示・操作を承認"]
    subgraph pc["Windows PC"]
        codex["Codex CLI"]
        ps["PowerShell"]
        liteapi["Lite用 HTTP API / daemon<br/>127.0.0.1:8000"]
        codex --> ps
        ps -->|"HTTP（PC内）"| liteapi
    end
    subgraph wireless["Reachy Mini Wireless 本体"]
        wirelessapi["Wireless用 HTTP API / daemon<br/>reachy-mini.local:8000"]
        control["モーターなどを制御"]
        wirelessapi --> control
    end
    lite["Reachy Mini Lite"]
    user --> codex
    liteapi -->|"USB"| lite
    ps -->|"HTTP（Wi-Fiネットワーク経由）"| wirelessapi
```

Lite用のAPIとdaemonはWindows PC上で動き、USB経由でLiteを制御します。Wireless用のAPIとdaemonは本体内で動きます。どちらもポート番号は8000ですが、接続先は異なります。最終的な機体操作ではWirelessへのSSHを使用していません。

この操作では、HTTP APIを提供する管理サーバーへの接続、daemon内部の制御開始、機体が通常姿勢へ動くことを区別します。`wake_up`は機体を通常姿勢に動かす指定です。実行環境と管理サーバーの準備は[実行環境](docs/environment.md)にまとめました。

| 対象 | PowerShellからの接続先 |
|---|---|
| Lite | `http://127.0.0.1:8000`（Windows PC自身） |
| Wireless | `http://reachy-mini.local:8000`（Wireless本体） |

## この資料で使う用語

| 用語 | この資料での意味 |
|---|---|
| 起動動作（wake-up／起床） | ロボットが頭とアンテナを動かし、通常の姿勢になることです。 |
| 収納 | ロボットが頭とアンテナを収納用の姿勢に戻すことです。 |
| daemon（デーモン） | ロボットの動作や状態を管理する制御プログラムです。 |
| HTTP API | PCからロボットの状態を調べたり、動作を指示したりするための窓口です。 |
| GET | 状態を問い合わせる要求です。 |
| POST | 起動動作や停止などの処理を依頼する要求です。 |

APIが要求を受け付けたことと、ロボットが実際に動いたことは、それぞれ確認する必要があります。そのため、この実験ではAPIの応答に加え、実験者による姿勢の確認を成功の条件としました。

## Codex CLIによる操作

実験では、まずWirelessを起動し、次にLiteを起動しました。終了時には、Lite、Wirelessの順に収納して停止しました。

Codexがコマンドを実行する前に、実験者はその内容とロボットへの影響を確認し、操作ごとに承認しました。コマンドを送った後は、CodexがAPIの応答を確認しました。実験者は、それとは別に、ロボットが目的の姿勢になったことと、異音や引っかかりがないことを確認しました。

状態確認、コマンドの提示、承認、API実行、実機観察の進め方は[Codex CLIとの協働手順](docs/codex-workflow.md)を参照してください。

両機とも起動動作と収納に成功しました。ただし、今回停止したのはロボットを制御するプログラムです。本体の電源スイッチは操作していません。

## 資料の読み方

2体を順に操作するときは「2体の操作順序」を参照してください。各機の操作手順には、状態確認と起動・収納・停止のコマンドをまとめています。

| ファイル | 内容 |
|---|---|
| [2体の操作順序](pair-start-stop.md) | Wireless→Liteの起動と、Lite→Wirelessの収納・停止 |
| [Wirelessの操作手順](wireless-operations.md) | Wirelessの状態確認、起動、収納・停止のコマンド |
| [Liteの操作手順](lite-operations.md) | Liteの状態確認、起動、収納・停止のコマンド |
| [実行環境](docs/environment.md) | 2体の配置、Python・SDK・daemonの実行環境と接続確認 |
| [Codex CLIとの協働手順](docs/codex-workflow.md) | 人間・ChatGPT・Codexの役割と承認・実機確認の流れ |

ここに記載した状態は、実験当時のものです。再実行するときは、その時点の状態を確認してください。

## 今回の成果と公式SDKとの関係

今回の起動・収納・停止実験のために、独自のプログラムは開発していません。このリポジトリでは、公式APIを呼び出すPowerShellコマンド、その実行手順、実行環境、協働方法、実機で確認した結果を公開します。元の開発プロジェクトにある別目的のコードは、今回の操作には使用していません。

APIとdaemonは、Pollen Roboticsが提供する公式SDK [pollen-robotics/reachy_mini](https://github.com/pollen-robotics/reachy_mini) の機能です。実験時にはSDK/daemon 1.13.0を使用しました。仕様の参照先は、公式の[REST API資料](https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/API/rest-api.mdx)と[daemonの実装](https://github.com/pollen-robotics/reachy_mini/blob/main/src/reachy_mini/daemon/daemon.py)です。リンク先のmainブランチは更新されるため、実験時の仕様と異なる場合があります。

Wirelessの起動動作は公式HTTP APIで成功しました。最終操作ではSSHを使用していません。また、両機の起動・収納・停止にOpenAI APIキーや有料APIは必要ありませんでした。2体が会話や撮影を連携して行う実験については、この資料では扱いません。

[デモ動画](https://youtu.be/xGLIWfysiMY)では、Codex CLIによる操作と実機の動きを示しています。今回確認したのは2体の順次起動・停止であり、自律協調・同期制御や音声・撮影を組み合わせた実機の一往復は検証していません。

## 再実行する際の注意

コマンドはWindows PowerShell用です。Wirelessの接続先は本体の管理API、Liteの接続先はWindows PC上の管理APIです。各機の手順に記載した接続先を確認してください。

実験者は、頭とアンテナの周囲から手や物を離してください。Codexに操作を任せる場合も、状態の問い合わせを含め、操作ごとにコマンドと影響を確認して承認してください。異音、引っかかり、過負荷、想定外の動きが生じた場合は、追加の動作を指示せず、状態を確認してください。


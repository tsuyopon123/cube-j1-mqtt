# Cube J1 MQTT

本プロジェクトは、2025年3月31日にサービスを終了した「NextDrive Cube J1」を活用し、Home Assistant の MQTT デバイスとして利用するためのツールです。

Cube J1 に内蔵されている Wi-SUN モジュール（BP35C0）を利用して、スマートメーター（B ルート）から各種計測値を定期的に取得し、MQTT 経由で Home Assistant へ送信します。

> [!WARNING]
> 本ツールの利用により、機器の動作不良、ネットワーク上のセキュリティリスク等が生じる可能性があります。
> 内容を十分に理解したうえで、利用者ご自身の責任で管理・運用してください。
> 本ツールの利用によって生じたいかなる損害についても、作成者および関係者は責任を負いません。

## 概要

Cube J1 上で専用の MQTT ブリッジプログラム（`mqtt_bridge.py`）が常駐稼働し、スマートメーターのデータを継続的に取得・送信します。
また、Home Assistant の「MQTT 自動検出（MQTT Auto Discovery）」に対応しているため、接続設定を済ませるだけでダッシュボードにセンサーが自動的に登録されます。

> [!NOTE]
> 本ツールを利用する前に、Home Assistant 側で MQTT ブローカー（Mosquitto broker など）の導入および MQTT 統合の設定を完了させておいてください。
> 参考: [MQTT 統合 — Home Assistant ドキュメント](https://www.home-assistant.io/integrations/mqtt/)

<img width="1032" height="504" alt="image" src="https://github.com/user-attachments/assets/daefc3f2-6c8a-416e-b433-1b45349d5f4f" />


## 取得できるセンサー

Home Assistant 上で以下のセンサーとしてデータを取り扱うことができます。

| センサー名 | ECHONET Lite EPC | 単位 | HA device_class |
|---|---|---|---|
| 瞬時電力 | E7 | W | power |
| 積算電力量（正方向） | E0 | kWh | energy (total_increasing) |
| 積算電力量（逆方向） | E3 | kWh | energy (total_increasing) |
| 瞬時電流 R相 | E8（上位2バイト） | A | current |
| 瞬時電流 T相 | E8（下位2バイト） | A | current |

※ 係数（EPC: D3）および積算電力量単位（EPC: E1）も自動で取得し、積算電力量の正確な kWh 換算に適用します。

## 導入方法

Cube J1 は USB メモリ内の特定ファイル構成を検出すると自動的にスクリプトを実行する仕組みを持っています。
リポジトリの内容をそのまま USB メモリ直下へコピーし起動するだけでセットアップが完了します。

### 手順

1. 本リポジトリをダウンロード（Clone）する。
2. `production_tool/config.json` を編集して、認証情報やネットワークの接続先を設定する（詳細は[config.json の設定](#configjson-の設定)を参照）。
3. `production_tool/wpa_supplicant.conf` を編集して、Wi-Fi の SSID とパスワードを設定する。
4. **FAT32 形式**でフォーマットされた USB メモリを用意し、`CubeJMTS.txt` と `production_tool/` ディレクトリを、USB メモリの直下（ルートディレクトリ）にコピーする。
5. Cube J1 に USB メモリを挿入し、電源を入れる。
6. 自動的にスクリプトが実行されます。セットアップが完了すると、**本体の LED が白色に 10 回点滅**します。また、Wi-Fi 接続が成功すると、LED は緑色に点灯します。

以降は自動的に MQTT ブリッジが起動し、スマートメーターからのデータ取得と Home Assistant への送信が開始されます。
※ 設定を再度変更したい場合は、USB メモリ内のファイルを編集して Cube J1 に挿入し、電源を再投入してください。

### config.json の設定

設定ファイル（`config.json`）の記入例および各項目の説明です。

```json
{
    "br_id":          "（スマートメーター B ルート認証 ID）",
    "br_pwd":         "（スマートメーター B ルートパスワード）",
    "mqtt_host":      "（Home Assistant の IP アドレス）",
    "mqtt_port":      1883,
    "mqtt_user":      "（MQTT ユーザー名）",
    "mqtt_pass":      "（MQTT パスワード）",
    "device_id":      "cubej1",
    "serial_port":    "/dev/ttyS1",
    "poll_interval":  60
}
```

| キー | 説明 |
|---|---|
| `br_id` | スマートメーターの B ルート認証 ID（32 文字） |
| `br_pwd` | スマートメーターの B ルートパスワード（12 文字） |
| `mqtt_host` | Home Assistant が動作しているサーバー・端末の IP アドレス |
| `mqtt_port` | MQTT ブローカーのポート番号（デフォルトは `1883`） |
| `mqtt_user` / `mqtt_pass` | MQTT ブローカーの認証情報（設定していない場合は空文字 `""` で可） |
| `device_id` | HA 上のデバイス識別子。 |
| `serial_port` | Wi-SUN モジュールのシリアルデバイス指定。通常は変更不要（`/dev/ttyS1`） |
| `poll_interval` | スマートメーターへデータを取得しに行くポーリング間隔（秒） |

## LED のステータス表示

Cube J1 の RGB LED は、動作状態に応じて以下のように発光・点滅します。

| 状態 | LED の動き |
|---|---|
| セットアップ完了時 | 白色で点滅（10回） |
| Wi-SUN コマンド送信中（SKSTACK） | 緑色と青色が交互に点滅（0.2 秒間隔） |
| PANA 接続待機中（SKJOIN） | 緑色と青色が交互に点滅（0.2 秒間隔） |
| データ取得・MQTTパブリッシュ中 | 青色で点灯 |

## システムの内部動作・仕様

技術要件等をメモとしてまとめます。

### セットアップ時の動作

USB メモリ挿入時に Cube J1 が自動実行するメインスクリプト（`production_tool`）は、以下の処理を順に行っています。

1. **ADB の TCP 有効化**: ポート `5555` で ADB 接続を受け付けるように設定
2. **Wi-Fi 設定**: `wpa_supplicant.conf` をシステムに配置してネットワークを再起動
3. **ブリッジプログラムの配置**: `config.json` と `mqtt_bridge.py` を `/data/local/` ディレクトリへコピー
4. **競合サービスの停止**: Wi-SUN モジュール（`/dev/ttyS1`）を占有してしまう既存サービス（`wisund`、`NDEcLiteAgent`）を停止し、以後の起動を無効化
5. **init サービスの登録**: 再起動後もプログラムが自動起動するよう、`mqtt_ha_bridge.rc` を `/system/etc/init/` へ配置
6. **ブリッジ即時起動**: `mqtt_ha_bridge` サービスとして `mqtt_bridge.py` を起動開始
7. **完了通知**: `led_effect.sh` を呼び出し、LED を点滅させてセットアップ完了を通知

### ファイル構成

```text
production_tool/
├── production_tool          # メインとなる自動実行セットアップスクリプト
├── mqtt_bridge.py           # Wi-SUN ↔ ECHONET Lite ↔ MQTT のブリッジプログラム本体
├── led_effect.sh            # RGB LED の点灯・点滅を制御するスクリプト
├── config.json              # 接続先などを指定する設定ファイル（要編集）
├── wpa_supplicant.conf      # Wi-Fi の接続先情報を指定する設定ファイル（要編集）
├── mqtt_ha_bridge.rc        # ブート時にブリッジを自動起動させるための init スクリプト
├── wisund_disabled.rc       # 標準の wisund サービスを無効化するための RC ファイル
└── ndeclite_disabled.rc     # 標準の NDEcLiteAgent を無効化するための RC ファイル
```

### 技術仕様詳細

- **実行環境**: Cube J1 上の Android 系 Linux（Python 2.7 にて動作）
- **依存ライブラリ**: Python 2.7 標準ライブラリのみを使用（`termios`, `socket`, `struct`, `select`, `json`, `threading` など）。`pyserial` や `paho-mqtt` 等の外部ライブラリは不要です。
- **シリアル通信**: `termios` にて raw モードを設定し、115200 bps で通信します。
- **MQTT 実装**: MQTT 3.1.1 の仕様に基づきソケット通信を用いて独自実装（QoS 0、TCP keepalive 対応、自動再接続機能あり）。
- **Wi-SUN 接続**: PAN スキャンを実行し、最も LQI（リンク品質）の良い PAN を自動選択します。
- **動作ログ**: ブリッジの動作ログは本体内の `/data/local/mqtt_bridge.log` に追記されます。

### MQTT トピック構造

| 用途 | トピック |
|---|---|
| HA auto-discovery | `homeassistant/sensor/{device_id}/{sensor_id}/config` |
| 瞬時電力 | `cubej/{device_id}/power` |
| 積算電力量（正方向） | `cubej/{device_id}/energy_forward` |
| 積算電力量（逆方向） | `cubej/{device_id}/energy_reverse` |
| 瞬時電流 R相 | `cubej/{device_id}/current_r` |
| 瞬時電流 T相 | `cubej/{device_id}/current_t` |

## 参考記事

Cube J1 のソフトウェア内部構造や、USB メモリを用いたスクリプト自動実行の仕組みについては、以下の記事で詳しく解説しています。

- [NextDrive Cube J1を分解せずにrootを取りたい！ - Zenn](https://zenn.dev/tsuyopon123/articles/cube-j1-root)

## トラブルシューティング

システムの状態や不具合の原因は、ADB 経由でログを確認することでデバッグが可能です。

```sh
# Cube J1 の IP アドレスに対し、ポート 5555 で ADB 接続
adb connect <Cube-J1 の IP アドレス>:5555

# 最新の動作ログを出力
adb shell cat /data/local/mqtt_bridge.log

# 実行中の Python プロセスを確認 (mqtt_bridge.py が動いているかどうか)
adb shell ps | grep python
```


# UZCP 1.0 仕様書

**Universal Zeta Coffee Protocol — Specification 1.0.1 (Draft)**

---

## 1. 概要

UZCP（Universal Zeta Coffee Protocol）は、コーヒー焙煎の完全自動化を目的としたオープンなJSONベースの通信プロトコルです。コントローラー、焙煎機、温度計、ロボットアームなど、焙煎プロセスに関わるすべてのデバイス間の相互運用性を実現します。

### 設計思想

* **シンプル** — JSONテキストベース。あらゆる言語・環境で実装可能
* **双方向** — コントローラーと焙煎デバイス間の対称的な通信
* **トランスポート非依存** — Wi-Fi、USB、Bluetooth、シリアル通信に対応
* **拡張性** — `x_` プレフィックスを使ったカスタムフィールドで互換性を壊さず拡張可能
* **自動化対応** — 手動操作から完全自律ロボット焙煎まで一つのプロトコルで対応

---

## 2. 用語定義

| 用語 | 定義 |
| --- | --- |
| Controller（コントローラー） | 焙煎を制御・監視するホスト（PC、スマートフォン、UZU ROASTERなど） |
| Device（デバイス） | 焙煎機、温度計、ロボットなどのエンドポイントデバイス |
| Profile（プロファイル） | 時間に対する目標温度曲線を定義した焙煎プロファイル |
| RoR | Rate of Rise（昇温速度）— ℃/minで表す温度上昇率 |
| Charge（投入） | 生豆をドラムに投入すること |
| FC | First Crack（1ハゼ） |
| SC | Second Crack（2ハゼ） |
| Drop（排出） | 焙煎豆を排出すること（焙煎終了） |

---

## 3. トランスポート層

UZCPはトランスポート非依存です。各実装は以下のトランスポートのうち少なくとも1つをサポートしなければなりません（MUST）。

### 3.1 Wi-Fi（HTTP REST）

```
Base URL: http://{device_ip}/uzcp/v1/
Content-Type: application/json
```

| メソッド | エンドポイント | 説明 |
| --- | --- | --- |
| GET | `/status` | デバイスステータス取得 |
| GET | `/telemetry` | 最新テレメトリのスナップショット取得 |
| POST | `/command` | コマンド送信 |
| GET | `/profile` | 現在のプロファイル取得 |
| POST | `/profile` | デバイスへのプロファイル書き込み |

### 3.2 Wi-Fi（WebSocket）

```
ws://{device_ip}/uzcp/v1/ws
```

接続後、双方がリアルタイムでメッセージを交換します。テレメトリのストリーミングに推奨。

### 3.3 USB / シリアル

```
ボーレート: 115200
改行コード: \n (LF)
エンコーディング: UTF-8
```

各メッセージはLFで終端された1行のJSONオブジェクトとします。

### 3.4 Bluetooth（BLE）

* サービスUUID: `UZCP-SERVICE-UUID`（実装者が定義）
* キャラクタリスティック: TX（Controller → Device）/ RX（Device → Controller）
* MTU: 1パケット最大512バイト。超過する場合は複数パケットに分割

---

## 4. メッセージ構造

すべてのUZCPメッセージは以下の共通ヘッダーを持ちます。

```json
{
  "uzcp": "1.0",
  "type": "<message_type>",
  "id": "<uuid_or_sequential_int>",
  "ts": 1700000000.123,
  "src": "<sender_id>",
  "dst": "<receiver_id>"
}
```

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `uzcp` | string | ✅ | プロトコルバージョン（例: `"1.0"`） |
| `type` | string | ✅ | メッセージタイプ（後述） |
| `id` | string | ✅ | メッセージ識別子（UUID推奨） |
| `ts` | float | ✅ | UNIXタイムスタンプ（秒、小数点以下可） |
| `src` | string | ✅ | 送信元デバイスID |
| `dst` | string | ✅ | 宛先デバイスID（`"*"` はブロードキャスト） |

---

## 4.1 接続ハンドシェイク *(1.0.1 新規追加)*

接続確立後、コントローラーは運用コマンドを送信する前に以下のハンドシェイクシーケンスを必ず実行しなければなりません（MUST）。

```
Controller                      Device
    |                               |
    |-- capability_request -------> |
    |<-- capability_response ------ |
    |                               |
    |-- set_time -----------------> |
    |<-- ack ---------------------- |
    |                               |
    |-- set_telemetry_interval ----> |
    |<-- ack ---------------------- |
    |                               |
    |   （ここからテレメトリ開始）      |
```

1. **capability_request / capability_response** — デバイスのサポート内容を問い合わせる
2. **set_time** — デバイスの時刻を同期する（RTCなしデバイスに必須）
3. **set_telemetry_interval** — テレメトリ送信間隔を指定する。デバイスはこれを受信して初めてテレメトリの送信を開始する

---

## 5. メッセージタイプ

### 5.1 telemetry（Device → Controller）

デバイスからリアルタイムの焙煎状態をストリーム送信します。

> **注意:** デバイスは `set_telemetry_interval` コマンドを受信するまでテレメトリを送信してはなりません（MUST NOT）。

```json
{
  "uzcp": "1.0",
  "type": "telemetry",
  "id": "a1b2c3d4",
  "ts": 1700000060.5,
  "src": "roaster-01",
  "dst": "*",
  "data": {
    "bt": 185.3,
    "et": 210.7,
    "ror": 8.2,
    "elapsed": 360,
    "phase": "maillard",
    "drum_rpm": 60,
    "airflow": 75,
    "heater": 80,
    "fc_detected": false,
    "sc_detected": false
  }
}
```

| フィールド | 型 | 単位 | 説明 |
| --- | --- | --- | --- |
| `bt` | float | ℃ | 豆温度（Bean Temperature） |
| `et` | float | ℃ | 排気温度（Environmental Temperature） |
| `ror` | float | ℃/min | 昇温速度（Rate of Rise） |
| `elapsed` | int | 秒 | 焙煎開始からの経過秒数 |
| `phase` | string | — | 現在の焙煎フェーズ（後述） |
| `drum_rpm` | int | rpm | ドラム回転数（対応デバイスのみ） |
| `airflow` | int | % | 風量（0〜100） |
| `heater` | int | % | ヒーター出力（0〜100） |
| `fc_detected` | bool | — | 1ハゼ検知フラグ |
| `sc_detected` | bool | — | 2ハゼ検知フラグ |

**フェーズ定義:**

| phase | 必須/任意 | 説明 |
| --- | --- | --- |
| `idle` | ✅ MUST | 待機中 |
| `roasting` | ✅ MUST | 焙煎中（フェーズ不明時のフォールバック） |
| `preheat` | 任意 | 予熱中 |
| `charge` | 任意 | 生豆投入直後 |
| `drying` | 任意 | 乾燥フェーズ |
| `maillard` | 任意 | メイラード反応フェーズ |
| `fc` | 任意 | 1ハゼ中 |
| `development` | 任意 | 発展フェーズ（1ハゼ〜排出） |
| `sc` | 任意 | 2ハゼ中 |
| `drop` | 任意 | 豆排出中 |
| `cooling` | 任意 | 冷却中 |

> `idle` と `roasting` 以外のサポートフェーズは `capability_response` で申告しなければなりません（MUST）。

---

### 5.2 command（Controller → Device）

デバイスに制御コマンドを送信します。

```json
{
  "uzcp": "1.0",
  "type": "command",
  "id": "b2c3d4e5",
  "ts": 1700000065.0,
  "src": "controller-01",
  "dst": "roaster-01",
  "cmd": "set_heater",
  "params": {
    "value": 85
  }
}
```

**コマンド一覧:**

| cmd | params | 説明 |
| --- | --- | --- |
| `start` | `{ "profile_id": "..." }` | 焙煎開始 |
| `stop` | — | 緊急停止 |
| `drop` | — | 豆排出トリガー |
| `set_heater` | `{ "value": 0–100 }` | ヒーター出力設定（%） |
| `set_airflow` | `{ "value": 0–100 }` | 風量設定（%） |
| `set_drum_rpm` | `{ "value": 0–200 }` | ドラム回転数設定 |
| `set_profile` | `{ "profile_id": "..." }` | アクティブプロファイル切替 |
| `charge` | `{ "weight_g": 200 }` | 生豆投入（ロボット用） |
| `preheat` | `{ "target_et": 200 }` | 目標排気温度まで予熱開始 |
| `set_time` *(NEW)* | `{ "ts": 1700000000.0, "tolerance_ms": 1000 }` | デバイス時刻をUNIXタイムで同期 |
| `set_telemetry_interval` *(NEW)* | `{ "interval_ms": 1000 }` | テレメトリ送信間隔設定。受信後にテレメトリ開始 |
| `custom` | `{ "key": "...", "value": "..." }` | カスタムコマンド |

#### set_time *(1.0.1 新規追加)*

RTCを持たないデバイスはコントローラーによる時刻同期に依存します。コントローラーはCapabilityハンドシェイク直後に `set_time` を送信するべきであり（SHOULD）、ドリフト補正のために定期的に再送してもよい（MAY）。

```json
{
  "uzcp": "1.0",
  "type": "command",
  "id": "...",
  "ts": 1700000000.0,
  "src": "controller-01",
  "dst": "roaster-01",
  "cmd": "set_time",
  "params": {
    "ts": 1700000000.0,
    "tolerance_ms": 1000
  }
}
```

| フィールド | 説明 |
| --- | --- |
| `ts` | 現在のUNIXタイムスタンプ（秒、float） |
| `tolerance_ms` | 許容時刻誤差（ミリ秒、デフォルト: 1000） |

#### set_telemetry_interval *(1.0.1 新規追加)*

デバイスはこのコマンドを受信するまでテレメトリを送信してはなりません（MUST NOT）。`interval_ms` を `0` に設定するとテレメトリを一時停止できます。

```json
{
  "cmd": "set_telemetry_interval",
  "params": {
    "interval_ms": 1000
  }
}
```

---

### 5.3 ack（Device → Controller）

コマンドの受信確認を返します。

```json
{
  "uzcp": "1.0",
  "type": "ack",
  "id": "c3d4e5f6",
  "ts": 1700000065.1,
  "src": "roaster-01",
  "dst": "controller-01",
  "ref_id": "b2c3d4e5",
  "status": "ok",
  "message": ""
}
```

| status | 説明 |
| --- | --- |
| `ok` | コマンド受理 |
| `error` | エラー発生（`message` に詳細） |
| `unsupported` | このデバイスでは未サポートのコマンド |

---

### 5.4 profile（双方向）

コントローラーとデバイス間で焙煎プロファイルを転送します。

```json
{
  "uzcp": "1.0",
  "type": "profile",
  "id": "d4e5f6a7",
  "ts": 1700000000.0,
  "src": "controller-01",
  "dst": "roaster-01",
  "profile": {
    "id": "ethiopia-natural-light",
    "name": "Ethiopia Natural Light Roast",
    "version": "1.0",
    "charge_temp_et": 200,
    "charge_weight_g": 200,
    "steps": [
      { "elapsed": 0,   "bt_target": 80,  "heater": 100, "airflow": 50, "drum_rpm": 60 },
      { "elapsed": 60,  "bt_target": 110, "heater": 90,  "airflow": 50, "drum_rpm": 60 },
      { "elapsed": 180, "bt_target": 150, "heater": 80,  "airflow": 60, "drum_rpm": 60 },
      { "elapsed": 300, "bt_target": 175, "heater": 70,  "airflow": 70, "drum_rpm": 60 },
      { "elapsed": 420, "bt_target": 195, "heater": 60,  "airflow": 80, "drum_rpm": 60 },
      { "elapsed": 480, "bt_target": 200, "heater": 50,  "airflow": 90, "drum_rpm": 60, "event": "drop" }
    ],
    "fc_auto_detect": true,
    "drop_bt": 200,
    "drop_ror": 3.0
  }
}
```

---

### 5.5 status（Device → Controller）

デバイスの現在の状態と能力を返します。

```json
{
  "uzcp": "1.0",
  "type": "status",
  "id": "e5f6a7b8",
  "ts": 1700000000.0,
  "src": "roaster-01",
  "dst": "controller-01",
  "device": {
    "id": "roaster-01",
    "name": "UZU ROASTER UZ-01",
    "firmware": "3.1.0",
    "uzcp_version": "1.0",
    "capabilities": ["telemetry", "profile", "set_heater", "set_airflow", "fc_detection"],
    "state": "idle",
    "uptime": 3600
  }
}
```

---

### 5.6 event（Device → Controller）

重要な焙煎イベントをコントローラーに通知します。

```json
{
  "uzcp": "1.0",
  "type": "event",
  "id": "f6a7b8c9",
  "ts": 1700000370.0,
  "src": "roaster-01",
  "dst": "*",
  "event": "fc",
  "data": {
    "bt": 196.2,
    "et": 218.5,
    "elapsed": 370
  }
}
```

**イベント一覧:**

| event | 説明 |
| --- | --- |
| `charge` | 生豆投入 |
| `fc` | 1ハゼ検知 |
| `sc` | 2ハゼ検知 |
| `drop` | 豆排出 |
| `alarm` | 異常検知（過温度など） |
| `profile_end` | プロファイルシーケンス完了 |
| `cooling_done` | 冷却サイクル完了 |

---

### 5.7 capability_request / capability_response *(1.0.1 新規追加)*

コントローラーがデバイスのサポート機能を問い合わせます。このハンドシェイクは接続確立直後、運用コマンド送信前に必ず実行しなければなりません（MUST）。

**リクエスト（Controller → Device）:**

```json
{
  "uzcp": "1.0",
  "type": "capability_request",
  "id": "a0b1c2d3",
  "ts": 1700000000.0,
  "src": "controller-01",
  "dst": "roaster-01"
}
```

**レスポンス（Device → Controller）:**

```json
{
  "uzcp": "1.0",
  "type": "capability_response",
  "id": "b1c2d3e4",
  "ts": 1700000000.1,
  "src": "roaster-01",
  "dst": "controller-01",
  "ref_id": "a0b1c2d3",
  "capabilities": {
    "commands": ["start", "stop", "set_heater", "set_airflow", "set_time", "set_telemetry_interval"],
    "events": ["fc", "sc", "drop", "alarm"],
    "telemetry_fields": ["bt", "et", "ror", "elapsed", "phase", "heater", "airflow"],
    "phases": ["idle", "roasting", "drying", "maillard", "fc", "development", "drop", "cooling"]
  }
}
```

| フィールド | 説明 |
| --- | --- |
| `commands` | サポートする `cmd` 値の一覧 |
| `events` | サポートする `event` 値の一覧 |
| `telemetry_fields` | `telemetry.data` に含まれるフィールドの一覧 |
| `phases` | サポートする `phase` 値の一覧（`idle` と `roasting` を含むこと） |

---

## 6. ロボット焙煎拡張

完全自動化・ロボット焙煎のための拡張仕様です。

### 6.1 robot\_status（Robot → Controller）

```json
{
  "uzcp": "1.0",
  "type": "robot_status",
  "id": "a7b8c9d0",
  "ts": 1700000000.0,
  "src": "robot-01",
  "dst": "controller-01",
  "robot": {
    "state": "idle",
    "hopper_weight_g": 1500,
    "last_charge_g": 200,
    "cycles_today": 5
  }
}
```

### 6.2 ロボットコマンド

| cmd | params | 説明 |
| --- | --- | --- |
| `charge` | `{ "weight_g": 200 }` | 指定重量の生豆を投入 |
| `collect` | `{ "bin": "light" }` | 指定ビンに焙煎豆を回収 |
| `clean` | — | ドラムクリーニングシーケンス実行 |
| `standby` | — | ロボットをスタンバイ位置に移動 |

---

## 7. エラー処理

### 7.1 エラーコード

| コード | 説明 |
| --- | --- |
| `E001` | 不明なメッセージタイプ |
| `E002` | 必須フィールド不足 |
| `E003` | 値が範囲外 |
| `E004` | デバイスビジー |
| `E005` | 無効なプロファイル |
| `E006` | 過温度アラーム |
| `E007` | 通信タイムアウト |
| `E999` | その他・不明なエラー |

```json
{
  "uzcp": "1.0",
  "type": "ack",
  "id": "...",
  "ts": 1700000000.0,
  "src": "roaster-01",
  "dst": "controller-01",
  "ref_id": "...",
  "status": "error",
  "error_code": "E003",
  "message": "heater value out of range: 150"
}
```

---

## 8. 拡張フィールド

`x_` プレフィックスを持つフィールドはカスタム用途に予約されています。標準フィールドと衝突してはならず（MUST NOT）、プロトコルの互換性を壊してはなりません（MUST NOT）。

`x_` フィールドが将来のバージョンで正式フィールドとして採用された場合、`x_` プレフィックスを外さなければなりません（MUST）。後方互換性のため、移行期間中は `x_` 付きの名前と正式名を併用してもよく（MAY）、依存システムの移行完了後は削除するべきです（SHOULD）。

```json
{
  "x_roast_level": "light",
  "x_bean_origin": "Ethiopia Yirgacheffe",
  "x_operator": "robot-arm-01"
}
```

---

## 9. バージョニング

| バージョン | 内容 |
| --- | --- |
| 1.0.0 | 初回リリース。テレメトリ・コマンド・プロファイル・イベント・ロボット拡張を含む |
| 1.0.1 | `set_time`、`set_telemetry_interval`、`capability_request/response` を追加。フェーズ必須要件（`idle`・`roasting`）を明確化。`x_` プレフィックス昇格ポリシーを正式化 |

---

## 10. 適合要件

* **MUST（必須）**: `telemetry`、`command`、`ack`、`status`、`capability_request`、`capability_response`
* **MUST（必須）コマンド**: `set_time`、`set_telemetry_interval`
* **MUST（必須）フェーズ**: `idle`、`roasting`
* **SHOULD（推奨）**: `event`、`profile`
* **MAY（任意）**: ロボット拡張、`x_` カスタムフィールド

UZCP 1.0 適合を謳うデバイスは、MUSTレベルのメッセージタイプおよびコマンドをすべて実装しなければなりません。

---

## 11. コントリビューション

UZCPはオープンプロトコルです。新しいメッセージタイプ、コマンド、拡張に関する提案はGitHubリポジトリで歓迎します。

* 仕様リポジトリ: [github.com/uzuuzuhonpo/uzcp](https://github.com/uzuuzuhonpo/uzcp)
* Issues & Discussions: コミュニティに公開

---

*UZCP 1.0.1 (Draft) — Designed by Uzuuzu Coffee Roastery*  
*UZU ROASTERは世界初のUZCP対応デバイスです。*


# UZCP 1.0 Specification

**Universal Zeta Coffee Protocol — Specification 1.0.1 (Draft)**

---

## 1. Overview

UZCP (Universal Zeta Coffee Protocol) is an open, JSON-based communication protocol designed for full automation of coffee roasting. It enables interoperability between all devices involved in the roasting process — controllers, roasters, thermometers, robot arms, and more.

### Design Principles

* **Simple** — JSON text-based; implementable in any language or environment
* **Bidirectional** — Symmetric communication between controllers and roasting devices
* **Transport-agnostic** — Supports Wi-Fi, USB, Bluetooth, and serial connections
* **Extensible** — Custom fields freely added using the `x_` prefix without breaking compatibility
* **Automation-ready** — One protocol from manual operation to fully autonomous robot roasting

---

## 2. Terminology

| Term | Definition |
| --- | --- |
| Controller | The host that controls and monitors the roast (PC, smartphone, UZU ROASTER, etc.) |
| Device | An endpoint device such as a roaster, thermometer, or robot |
| Profile | A roast profile defining a target temperature curve over time |
| RoR | Rate of Rise — temperature increase rate in ℃/min |
| Charge | Loading green beans into the drum |
| FC | First Crack |
| SC | Second Crack |
| Drop | Ejecting the roasted beans (end of roast) |

---

## 3. Transport Layer

UZCP is transport-agnostic. Each implementation MUST support at least one of the following transports.

### 3.1 Wi-Fi (HTTP REST)

```
Base URL: http://{device_ip}/uzcp/v1/
Content-Type: application/json
```

| Method | Endpoint | Description |
| --- | --- | --- |
| GET | `/status` | Get device status |
| GET | `/telemetry` | Get latest telemetry snapshot |
| POST | `/command` | Send a command |
| GET | `/profile` | Retrieve the current profile |
| POST | `/profile` | Write a profile to the device |

### 3.2 Wi-Fi (WebSocket)

```
ws://{device_ip}/uzcp/v1/ws
```

After connection, both sides exchange messages in real time. Recommended for telemetry streaming.

### 3.3 USB / Serial

```
Baud rate : 115200
Line ending: \n (LF)
Encoding  : UTF-8
```

Each message is one JSON object per line, terminated by LF.

### 3.4 Bluetooth (BLE)

* Service UUID: `UZCP-SERVICE-UUID` (defined by implementer)
* Characteristics: TX (Controller → Device) / RX (Device → Controller)
* MTU: up to 512 bytes per packet; split into multiple packets if exceeded

---

## 4. Message Structure

Every UZCP message shares the following common header.

```json
{
  "uzcp": "1.0",
  "type": "<message_type>",
  "id": "<uuid_or_sequential_int>",
  "ts": 1700000000.123,
  "src": "<sender_id>",
  "dst": "<receiver_id>"
}
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `uzcp` | string | ✅ | Protocol version (e.g. `"1.0"`) |
| `type` | string | ✅ | Message type (see below) |
| `id` | string | ✅ | Message identifier (UUID recommended) |
| `ts` | float | ✅ | UNIX timestamp in seconds (fractional allowed) |
| `src` | string | ✅ | Sender device ID |
| `dst` | string | ✅ | Destination device ID (`"*"` for broadcast) |

---

## 4.1 Connection Handshake *(NEW in 1.0.1)*

Upon establishing a connection, the controller MUST perform the following handshake sequence before sending any operational commands.

```
Controller                      Device
    |                               |
    |-- capability_request -------> |
    |<-- capability_response ------ |
    |                               |
    |-- set_time -----------------> |
    |<-- ack ---------------------- |
    |                               |
    |-- set_telemetry_interval ----> |
    |<-- ack ---------------------- |
    |                               |
    |   (telemetry starts now)      |
```

1. **capability_request / capability_response** — Controller queries what the device supports
2. **set_time** — Controller synchronizes device clock (required for devices without RTC)
3. **set_telemetry_interval** — Controller specifies telemetry interval; device begins streaming only after receiving this

---

## 5. Message Types

### 5.1 telemetry (Device → Controller)

Streams real-time roasting state from the device.

> **Note:** The device MUST NOT send telemetry until it receives a `set_telemetry_interval` command.

```json
{
  "uzcp": "1.0",
  "type": "telemetry",
  "id": "a1b2c3d4",
  "ts": 1700000060.5,
  "src": "roaster-01",
  "dst": "*",
  "data": {
    "bt": 185.3,
    "et": 210.7,
    "ror": 8.2,
    "elapsed": 360,
    "phase": "maillard",
    "drum_rpm": 60,
    "airflow": 75,
    "heater": 80,
    "fc_detected": false,
    "sc_detected": false
  }
}
```

| Field | Type | Unit | Description |
| --- | --- | --- | --- |
| `bt` | float | ℃ | Bean Temperature |
| `et` | float | ℃ | Environmental (exhaust) Temperature |
| `ror` | float | ℃/min | Rate of Rise |
| `elapsed` | int | sec | Seconds since roast start |
| `phase` | string | — | Current roast phase (see below) |
| `drum_rpm` | int | rpm | Drum rotation speed (if supported) |
| `airflow` | int | % | Airflow level (0–100) |
| `heater` | int | % | Heater power (0–100) |
| `fc_detected` | bool | — | First Crack detection flag |
| `sc_detected` | bool | — | Second Crack detection flag |

**Phase Definitions:**

| phase | Required | Description |
| --- | --- | --- |
| `idle` | ✅ MUST | Standby |
| `roasting` | ✅ MUST | Generic roasting (fallback when specific phase unknown) |
| `preheat` | optional | Preheating |
| `charge` | optional | Immediately after green bean charge |
| `drying` | optional | Drying phase |
| `maillard` | optional | Maillard reaction phase |
| `fc` | optional | During First Crack |
| `development` | optional | Development phase (FC to Drop) |
| `sc` | optional | During Second Crack |
| `drop` | optional | Beans ejected |
| `cooling` | optional | Cooling |

> Supported phases beyond `idle` and `roasting` MUST be declared in `capability_response`.

---

### 5.2 command (Controller → Device)

Sends a control command to a device.

```json
{
  "uzcp": "1.0",
  "type": "command",
  "id": "b2c3d4e5",
  "ts": 1700000065.0,
  "src": "controller-01",
  "dst": "roaster-01",
  "cmd": "set_heater",
  "params": {
    "value": 85
  }
}
```

**Command Reference:**

| cmd | params | Description |
| --- | --- | --- |
| `start` | `{ "profile_id": "..." }` | Start roasting |
| `stop` | — | Emergency stop |
| `drop` | — | Trigger bean ejection |
| `set_heater` | `{ "value": 0–100 }` | Set heater power (%) |
| `set_airflow` | `{ "value": 0–100 }` | Set airflow level (%) |
| `set_drum_rpm` | `{ "value": 0–200 }` | Set drum rotation speed |
| `set_profile` | `{ "profile_id": "..." }` | Switch active profile |
| `charge` | `{ "weight_g": 200 }` | Charge green beans (robot use) |
| `preheat` | `{ "target_et": 200 }` | Begin preheat to target ET |
| `set_time` *(NEW)* | `{ "ts": 1700000000.0, "tolerance_ms": 1000 }` | Sync UNIX timestamp to device clock |
| `set_telemetry_interval` *(NEW)* | `{ "interval_ms": 1000 }` | Set telemetry interval; device starts sending after this |
| `custom` | `{ "key": "...", "value": "..." }` | Custom command |

#### set_time *(NEW in 1.0.1)*

Devices without an RTC rely on the controller for time synchronization. The controller SHOULD send `set_time` immediately after the capability handshake, and MAY re-send it periodically to correct drift.

```json
{
  "uzcp": "1.0",
  "type": "command",
  "id": "...",
  "ts": 1700000000.0,
  "src": "controller-01",
  "dst": "roaster-01",
  "cmd": "set_time",
  "params": {
    "ts": 1700000000.0,
    "tolerance_ms": 1000
  }
}
```

| Field | Description |
| --- | --- |
| `ts` | Current UNIX timestamp (seconds, float) |
| `tolerance_ms` | Acceptable clock error in milliseconds (default: 1000) |

#### set_telemetry_interval *(NEW in 1.0.1)*

The device MUST NOT send telemetry until this command is received. Setting `interval_ms` to `0` pauses telemetry.

```json
{
  "cmd": "set_telemetry_interval",
  "params": {
    "interval_ms": 1000
  }
}
```

---

### 5.3 ack (Device → Controller)

Acknowledges receipt of a command.

```json
{
  "uzcp": "1.0",
  "type": "ack",
  "id": "c3d4e5f6",
  "ts": 1700000065.1,
  "src": "roaster-01",
  "dst": "controller-01",
  "ref_id": "b2c3d4e5",
  "status": "ok",
  "message": ""
}
```

| status | Description |
| --- | --- |
| `ok` | Command accepted |
| `error` | Error occurred (`message` contains details) |
| `unsupported` | Command not supported by this device |

---

### 5.4 profile (Bidirectional)

Transfers a roast profile between controller and device.

```json
{
  "uzcp": "1.0",
  "type": "profile",
  "id": "d4e5f6a7",
  "ts": 1700000000.0,
  "src": "controller-01",
  "dst": "roaster-01",
  "profile": {
    "id": "ethiopia-natural-light",
    "name": "Ethiopia Natural Light Roast",
    "version": "1.0",
    "charge_temp_et": 200,
    "charge_weight_g": 200,
    "steps": [
      { "elapsed": 0,   "bt_target": 80,  "heater": 100, "airflow": 50, "drum_rpm": 60 },
      { "elapsed": 60,  "bt_target": 110, "heater": 90,  "airflow": 50, "drum_rpm": 60 },
      { "elapsed": 180, "bt_target": 150, "heater": 80,  "airflow": 60, "drum_rpm": 60 },
      { "elapsed": 300, "bt_target": 175, "heater": 70,  "airflow": 70, "drum_rpm": 60 },
      { "elapsed": 420, "bt_target": 195, "heater": 60,  "airflow": 80, "drum_rpm": 60 },
      { "elapsed": 480, "bt_target": 200, "heater": 50,  "airflow": 90, "drum_rpm": 60, "event": "drop" }
    ],
    "fc_auto_detect": true,
    "drop_bt": 200,
    "drop_ror": 3.0
  }
}
```

---

### 5.5 status (Device → Controller)

Returns the current state and capabilities of the device.

```json
{
  "uzcp": "1.0",
  "type": "status",
  "id": "e5f6a7b8",
  "ts": 1700000000.0,
  "src": "roaster-01",
  "dst": "controller-01",
  "device": {
    "id": "roaster-01",
    "name": "UZU ROASTER UZ-01",
    "firmware": "3.1.0",
    "uzcp_version": "1.0",
    "capabilities": ["telemetry", "profile", "set_heater", "set_airflow", "fc_detection"],
    "state": "idle",
    "uptime": 3600
  }
}
```

---

### 5.6 event (Device → Controller)

Notifies the controller of a significant roasting event.

```json
{
  "uzcp": "1.0",
  "type": "event",
  "id": "f6a7b8c9",
  "ts": 1700000370.0,
  "src": "roaster-01",
  "dst": "*",
  "event": "fc",
  "data": {
    "bt": 196.2,
    "et": 218.5,
    "elapsed": 370
  }
}
```

**Event Reference:**

| event | Description |
| --- | --- |
| `charge` | Green beans charged |
| `fc` | First Crack detected |
| `sc` | Second Crack detected |
| `drop` | Beans ejected |
| `alarm` | Anomaly detected (e.g. over-temperature) |
| `profile_end` | Profile sequence completed |
| `cooling_done` | Cooling cycle complete |

---

### 5.7 capability_request / capability_response *(NEW in 1.0.1)*

The controller queries the device's supported capabilities. This handshake MUST occur immediately after connection establishment, before any operational commands.

**Request (Controller → Device):**

```json
{
  "uzcp": "1.0",
  "type": "capability_request",
  "id": "a0b1c2d3",
  "ts": 1700000000.0,
  "src": "controller-01",
  "dst": "roaster-01"
}
```

**Response (Device → Controller):**

```json
{
  "uzcp": "1.0",
  "type": "capability_response",
  "id": "b1c2d3e4",
  "ts": 1700000000.1,
  "src": "roaster-01",
  "dst": "controller-01",
  "ref_id": "a0b1c2d3",
  "capabilities": {
    "commands": ["start", "stop", "set_heater", "set_airflow", "set_time", "set_telemetry_interval"],
    "events": ["fc", "sc", "drop", "alarm"],
    "telemetry_fields": ["bt", "et", "ror", "elapsed", "phase", "heater", "airflow"],
    "phases": ["idle", "roasting", "drying", "maillard", "fc", "development", "drop", "cooling"]
  }
}
```

| Field | Description |
| --- | --- |
| `commands` | List of supported `cmd` values |
| `events` | List of supported `event` values |
| `telemetry_fields` | List of fields included in `telemetry.data` |
| `phases` | List of supported `phase` values (MUST include `idle` and `roasting`) |

---

## 6. Robot Roasting Extension

Extended specification for full automation and robotic roasting.

### 6.1 robot\_status (Robot → Controller)

```json
{
  "uzcp": "1.0",
  "type": "robot_status",
  "id": "a7b8c9d0",
  "ts": 1700000000.0,
  "src": "robot-01",
  "dst": "controller-01",
  "robot": {
    "state": "idle",
    "hopper_weight_g": 1500,
    "last_charge_g": 200,
    "cycles_today": 5
  }
}
```

### 6.2 Robot Commands

| cmd | params | Description |
| --- | --- | --- |
| `charge` | `{ "weight_g": 200 }` | Charge specified weight of green beans |
| `collect` | `{ "bin": "light" }` | Collect roasted beans into the specified bin |
| `clean` | — | Run drum cleaning sequence |
| `standby` | — | Move robot to standby position |

---

## 7. Error Handling

### 7.1 Error Codes

| code | Description |
| --- | --- |
| `E001` | Unknown message type |
| `E002` | Missing required field |
| `E003` | Value out of range |
| `E004` | Device busy |
| `E005` | Invalid profile |
| `E006` | Over-temperature alarm |
| `E007` | Communication timeout |
| `E999` | Other / unspecified error |

```json
{
  "uzcp": "1.0",
  "type": "ack",
  "id": "...",
  "ts": 1700000000.0,
  "src": "roaster-01",
  "dst": "controller-01",
  "ref_id": "...",
  "status": "error",
  "error_code": "E003",
  "message": "heater value out of range: 150"
}
```

---

## 8. Extension Fields

Fields prefixed with `x_` are reserved for custom use. They MUST NOT conflict with standard fields and MUST NOT break protocol compatibility.

When an `x_`-prefixed field is adopted as a standard field in a future version, the `x_` prefix MUST be removed. For backward compatibility, implementations MAY support the `x_`-prefixed name alongside the standard name during a transition period, and SHOULD remove it once all dependent systems have migrated.

```json
{
  "x_roast_level": "light",
  "x_bean_origin": "Ethiopia Yirgacheffe",
  "x_operator": "robot-arm-01"
}
```

---

## 9. Versioning

| Version | Notes |
| --- | --- |
| 1.0.0 | Initial release. Core telemetry, commands, profiles, events, and robot extension. |
| 1.0.1 | Added `set_time`, `set_telemetry_interval`, `capability_request/response`. Clarified phase requirements (`idle`, `roasting` mandatory). Formalized `x_` promotion policy. |

---

## 10. Conformance Requirements

* **MUST** implement: `telemetry`, `command`, `ack`, `status`, `capability_request`, `capability_response`
* **MUST** support commands: `set_time`, `set_telemetry_interval`
* **MUST** support phases: `idle`, `roasting`
* **SHOULD** implement: `event`, `profile`
* **MAY** implement: Robot extension, `x_` custom fields

A device claiming UZCP 1.0 conformance MUST implement all MUST-level message types and commands.

---

## 11. Contributing

UZCP is an open protocol. Proposals for new message types, commands, or extensions are welcome via the GitHub repository.

* Spec Repository: [github.com/uzuuzuhonpo/uzcp](https://github.com/uzuuzuhonpo/uzcp)
* Issues & Discussions: open to the community

---

*UZCP 1.0.1 (Draft) — Designed by Uzuuzu Coffee Roastery*  
*UZU ROASTER is the world's first UZCP-compatible device.*
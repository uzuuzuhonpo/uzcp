# UZCP 1.0 仕様書
**Universal Zeta Coffee Protocol — Specification 1.0.0**

---

## 1. 概要

UZCP（Universal Zeta Coffee Protocol）は、コーヒー焙煎の完全自動化を目的とした、JSON ベースのオープン通信プロトコルです。コントローラー・焙煎機・温度計・ロボットアームなど、焙煎に関わるあらゆるデバイス間の相互運用性を実現します。

### 設計思想

- **シンプル** ― JSON テキストベースで、あらゆる言語・環境から実装可能
- **双方向** ― コントローラーと焙煎デバイス間で対等に通信
- **トランスポート非依存** ― Wi-Fi / USB / Bluetooth / シリアルすべてに対応
- **拡張可能** ― カスタムフィールドを `x_` プレフィックスで自由に追加可能
- **自動化対応** ― 手動操作からロボット焙煎まで同一プロトコルで対応

---

## 2. 用語定義

| 用語 | 説明 |
|------|------|
| Controller | 焙煎を制御・監視するホスト（PC / スマホ / UZU ROASTER 等） |
| Device | 焙煎機・温度計・ロボット等の末端デバイス |
| Profile | 焙煎プロファイル（時間と目標温度のカーブ） |
| RoR | Rate of Rise（温度上昇率 ℃/min） |
| Charge | 生豆投入 |
| FC | First Crack（1ハゼ） |
| SC | Second Crack（2ハゼ） |
| Drop | 排出（焙煎完了） |

---

## 3. トランスポート層

UZCP はトランスポートを選ばない。各実装は以下のいずれかに対応すること。

### 3.1 Wi-Fi（HTTP REST）

```
Base URL: http://{device_ip}/uzcp/v1/
Content-Type: application/json
```

- **GET** `/status` ― デバイス状態取得
- **GET** `/telemetry` ― 最新テレメトリ取得
- **POST** `/command` ― コマンド送信
- **GET** `/profile` ― プロファイル取得
- **POST** `/profile` ― プロファイル書き込み

### 3.2 Wi-Fi（WebSocket）

```
ws://{device_ip}/uzcp/v1/ws
```

接続後、双方向でリアルタイムにメッセージを送受信する。テレメトリのストリーミングに推奨。

### 3.3 USB / シリアル

```
Baud rate: 115200
改行コード: \n（LF）
エンコード: UTF-8
```

各メッセージは 1 行 1 JSON で改行終端。

### 3.4 Bluetooth (BLE)

- Service UUID: `UZCP-SERVICE-UUID`（実装者が定義）
- Characteristic: TX（Controller→Device）/ RX（Device→Controller）
- MTU: 最大 512 bytes / パケット。超過する場合は分割送信。

---

## 4. メッセージ構造

すべてのメッセージは以下の共通ヘッダを持つ。

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
|-----------|-----|------|------|
| `uzcp` | string | ✅ | プロトコルバージョン（例: `"1.0"`） |
| `type` | string | ✅ | メッセージ種別（下記参照） |
| `id` | string | ✅ | メッセージ識別子（UUID 推奨） |
| `ts` | float | ✅ | UNIX タイムスタンプ（秒、小数可） |
| `src` | string | ✅ | 送信元デバイス ID |
| `dst` | string | ✅ | 宛先デバイス ID（ブロードキャストは `"*"`） |

---

## 5. メッセージ種別

### 5.1 telemetry（Device → Controller）

焙煎中の状態をリアルタイムで送信する。

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
|-----------|-----|------|------|
| `bt` | float | ℃ | Bean Temperature（豆温度） |
| `et` | float | ℃ | Environmental Temperature（排気温度） |
| `ror` | float | ℃/min | 温度上昇率 |
| `elapsed` | int | 秒 | 焙煎開始からの経過時間 |
| `phase` | string | ― | 焙煎フェーズ（下記参照） |
| `drum_rpm` | int | rpm | ドラム回転数（対応機器のみ） |
| `airflow` | int | % | 風量（0-100） |
| `heater` | int | % | 火力（0-100） |
| `fc_detected` | bool | ― | 1ハゼ検出フラグ |
| `sc_detected` | bool | ― | 2ハゼ検出フラグ |

**フェーズ定義：**

| phase | 説明 |
|-------|------|
| `idle` | 待機中 |
| `preheat` | 予熱中 |
| `charge` | 生豆投入直後 |
| `drying` | 乾燥フェーズ |
| `maillard` | メイラード反応フェーズ |
| `fc` | 1ハゼ中 |
| `development` | 発展フェーズ（FC〜Drop） |
| `sc` | 2ハゼ中 |
| `drop` | 排出済み |
| `cooling` | 冷却中 |

---

### 5.2 command（Controller → Device）

デバイスへの制御コマンドを送信する。

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

**コマンド一覧：**

| cmd | params | 説明 |
|-----|--------|------|
| `start` | `{ "profile_id": "..." }` | 焙煎開始 |
| `stop` | ― | 緊急停止 |
| `drop` | ― | 排出指示 |
| `set_heater` | `{ "value": 0-100 }` | 火力設定（%） |
| `set_airflow` | `{ "value": 0-100 }` | 風量設定（%） |
| `set_drum_rpm` | `{ "value": 0-200 }` | ドラム回転数設定 |
| `set_profile` | `{ "profile_id": "..." }` | プロファイル切替 |
| `charge` | `{ "weight_g": 200 }` | 生豆投入指示（ロボット用） |
| `preheat` | `{ "target_et": 200 }` | 予熱開始 |
| `custom` | `{ "key": "...", "value": "..." }` | カスタムコマンド |

---

### 5.3 ack（Device → Controller）

コマンドの受信確認。

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
|--------|------|
| `ok` | 正常受理 |
| `error` | エラー（`message` にエラー内容） |
| `unsupported` | 非対応コマンド |

---

### 5.4 profile（双方向）

焙煎プロファイルの送受信。

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

デバイスの現在状態・能力を返す。

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

焙煎中の重要イベントを通知する。

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

**イベント一覧：**

| event | 説明 |
|-------|------|
| `charge` | 生豆投入検出 |
| `fc` | 1ハゼ検出 |
| `sc` | 2ハゼ検出 |
| `drop` | 排出完了 |
| `alarm` | 異常検出（過温度等） |
| `profile_end` | プロファイル終了 |
| `cooling_done` | 冷却完了 |

---

## 6. ロボット焙煎拡張

完全自動化・ロボット焙煎向けの拡張仕様。

### 6.1 robot_status（Robot → Controller）

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

### 6.2 ロボット対応コマンド

| cmd | params | 説明 |
|-----|--------|------|
| `charge` | `{ "weight_g": 200 }` | 指定グラム数で生豆投入 |
| `collect` | `{ "bin": "light" }` | 指定ビンへ排出豆を回収 |
| `clean` | ― | ドラム清掃 |
| `standby` | ― | 待機状態へ移行 |

---

## 7. エラーハンドリング

### 7.1 エラーコード

| code | 説明 |
|------|------|
| `E001` | 不明なメッセージ種別 |
| `E002` | 必須フィールド欠如 |
| `E003` | 値が範囲外 |
| `E004` | デバイスビジー |
| `E005` | プロファイル不正 |
| `E006` | 過温度アラーム |
| `E007` | 通信タイムアウト |
| `E999` | その他エラー |

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

`x_` プレフィックスのフィールドは自由に追加可能。プロトコルの互換性を損なわない。

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
|-----------|------|
| 1.0.0 | 初版。基本テレメトリ・コマンド・プロファイル・ロボット拡張 |

---

## 10. 実装要件

- **MUST**（必須）: `telemetry`、`command`、`ack`、`status` への対応
- **SHOULD**（推奨）: `event`、`profile` への対応
- **MAY**（任意）: ロボット拡張、`x_` カスタムフィールド

---

*UZCP 1.0.0 — Designed by うずうず本舗*
*UZU ROASTER is the world's first UZCP-compatible device.*

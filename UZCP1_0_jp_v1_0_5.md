
# UZCP 1.0 仕様書

**Universal Zeta Coffee Protocol — Specification 1.0.5 (Draft)**

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
| Profile（プロファイル） | 焙煎レシピと実績データを統合したUZCPの中心的データオブジェクト。コンテキストに応じてレシピのみ・実績のみ・両方を含む |
| RoR | Rate of Rise（昇温速度）— ℃/minで表す温度上昇率 |
| Charge（投入） | 生豆をドラムに投入すること |
| FC | First Crack（1ハゼ） |
| SC | Second Crack（2ハゼ） |
| Drop（排出） | 焙煎豆を排出すること（焙煎終了） |

---

## 3. トランスポート層

UZCPはトランスポート非依存です。各実装は以下のトランスポートのうち少なくとも1つをサポートしなければなりません（MUST）。

### 3.0 メッセージデリミタ（全トランスポート共通）

**すべてのUZCPメッセージは改行文字（LF: `\n` / 0x0A）で終端しなければなりません（MUST）。**

#### シリアライゼーション規則
- JSONは **minify**（空白・改行削除）しなければなりません（MUST）
- pretty print（整形）されたJSONは禁止です（MUST NOT）
- エンコーディング: UTF-8
- 数値型（int, float）はJSON仕様に従いテキスト表現。バイナリエンコード（IEEE 754等）は使用しなません

#### 送信
1. JSONオブジェクトをシリアライズ（minify）
2. 末尾に `\n` を追加
3. UTF-8エンコード
4. トランスポート層で送信

例:
```
{"uzcp":"1.0","type":"telemetry","id":"a1b2","ts":1700000060.5,"src":"roaster-01","dst":"*","data":{"bt":185.3}}\n
```

#### 受信
1. バイトストリームをバッファに蓄積
2. `\n` を検出したらその前までを1メッセージとして抽出
3. JSONパース
4. バッファから削除、次のメッセージを待機

#### 複数メッセージ
複数のメッセージを連続送信する場合、各メッセージは `\n` で区切られます:
```
{"type":"telemetry",...}\n
{"type":"command",...}\n
{"type":"response",...}\n
```

#### 表示・デバッグ
メッセージは minify された状態で送受信されます。人間が読む場合（ログ表示、UI等）は、受信側アプリケーションが適宜 pretty print してもかまいません（MAY）。

#### エラー処理
- `\n` が来ない場合: タイムアウト（推奨: 5秒）
- JSONパース失敗: エラーログ出力、バッファクリア、次のメッセージを待機

---

### 3.1 Wi-Fi（HTTP REST）

```
Base URL: http://{device_ip}/uzcp/v1/
Content-Type: application/json
```

| メソッド | エンドポイント | 説明 |
| --- | --- | --- |
| GET | `/get_telemetry` | 最新テレメトリのスナップショット取得 |
| GET | `/get_event` | イベントの取得 |
| POST | `/command` | コマンド送信 |

**HTTPの場合:**
- リクエスト・レスポンスボディは改行で終端されたJSONです
- `Content-Length` ヘッダーで長さを指定します

### 3.2 Wi-Fi（WebSocket）

```
ws://{device_ip}/uzcp/v1/ws
```

接続後、双方がリアルタイムでメッセージを交換します。テレメトリのストリーミングに推奨。

**WebSocketの場合:**
- 各WebSocketフレームには改行で終端された1つ以上のJSONメッセージを含めることができます
- 1フレーム = 1メッセージが推奨ですが、複数メッセージをまとめて送信してもかまいません（MAY）

### 3.3 USB / シリアル

```
ボーレート: 115200
改行コード: \n (LF)
エンコーディング: UTF-8
```

各メッセージはLFで終端された1行のJSONです。

**実装例（受信側）:**
```cpp
String buffer = "";

void onSerialData() {
  while (Serial.available()) {
    char c = Serial.read();
    buffer += c;
    
    if (c == '\n') {
      parseUZCPMessage(buffer);
      buffer = "";
    }
  }
}
```

### 3.4 Bluetooth（BLE）

- **サービスUUID**: `UZCP-SERVICE-UUID`（実装者が定義）
- **キャラクタリスティック**: TX（Controller → Device）/ RX（Device → Controller）
- **MTU**: 1パケット最大512バイト（BLE 5.0）、247バイト（BLE 4.2）、20バイト（BLE 4.0）

**パケット分割:**
- メッセージがMTUを超える場合、複数パケットに分割して送信します
- 受信側は `\n` を検出するまでバッファに蓄積します
- MTU交渉（`requestMTU()`）を推奨します

**実装例（受信側）:**
```cpp
String buffer = "";

void onBLEWrite(BLECharacteristic* pCharacteristic) {
  std::string value = pCharacteristic->getValue();
  buffer += String(value.c_str());
  
  int pos;
  while ((pos = buffer.indexOf('\n')) != -1) {
    String json = buffer.substring(0, pos);
    parseUZCPMessage(json);
    buffer = buffer.substring(pos + 1);
  }
}
```

### 3.5 実装ガイドライン

UZCPはトランスポート非依存のプロトコルですが、異なる通信媒体やデバイス間での確実なデータ授受を保証するため、以下の実装ルールを定めます。

#### 3.5.1 メッセージの終端定義と断片化への対応 (MUST)

* **終端識別子の付与**: すべてのUZCPメッセージ（JSONデータ）の末尾には、必ず **ラインフィード（`\n`, `0x0A`）** を付与しなければなりません。
* **メッセージの再構成（バッファリング）**: 受信側は、終端識別子（`\n`）を受信するまで、物理層で分割されて届くパケットを一連のメッセージとして蓄積し、結合しなければなりません。
    * *補足：これにより、BLE（Bluetooth Low Energy）のMTU制限による分割や、シリアル通信におけるバッファの断片化が発生する環境においても、長大なJSONデータの整合性を保証します。*

#### 3.5.2 推奨される物理接続クラス (SHOULD)

* **USB接続**: 
    * デバイスをUSB経由で接続する場合、**USB-CDC（Communication Device Class / 仮想シリアルポート）** の使用を強く推奨します。
    * これにより、OS標準のドライバによる動作を可能にし、ボーレート等の物理パラメータ設定に依存しないストリーム通信を実現します。
* **Bluetooth接続**:
    * BLEを用いる場合は、NotifyまたはWrite特性を利用し、3.5.1項の規定に従って `\n` を終端としたストリームとしてデータを送受信してください。

#### 3.5.3 通信モデルの種別と運用 (SHOULD)

通信経路の特性に応じて、以下のいずれかのモデルを採用してください。

* **双方向ストリーミング型 (Serial, Wi-Fi TCP, USB-CDC 等)**:
    * コントローラーおよびデバイス双方が、任意のタイミングでメッセージを送信可能です。イベント発生時にはデバイス側から即時の通知（Push）を行うことを推奨します。
* **ポーリング型 (USB Bulk, HTTP 等)**:
    * デバイス側からの自発的な通知が困難なトランスポート環境においては、コントローラーが定期的に（推奨500ms〜1000ms間隔）`get_telemetry` コマンド等を送信し、デバイスの状態を明示的に取得（Pull）しなければなりません。

---

## 4. メッセージ構造

すべてのUZCPメッセージは以下の共通エンベロープを持ちます。

```json
{
  "uzcp": "1.0",
  "type": "<message_type>",
  "id": "<uuid_or_sequential_id>",
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
| `ts` | float | ✅ | タイムスタンプ（秒、小数点以下可）。`set_time` でコントローラーが設定した基準時刻からの経過秒数 |
| `src` | string | ✅ | 送信元デバイスID |
| `dst` | string | ✅ | 宛先デバイスID（`"*"` はブロードキャスト） |

### メッセージタイプ一覧

| type | 方向 | 説明 |
| --- | --- | --- |
| `telemetry` | Device → Controller | リアルタイム焙煎状態のストリーム |
| `command` | Controller → Device | 制御コマンド |
| `response` | Device → Controller | コマンドへの応答 |
| `event` | Device → Controller | 重要焙煎イベントの通知 |
| `roast_log` | 双方向 / ファイル保存 | 焙煎結果の記録・転送 |
| `capability_request` | Controller → Device | 機能問い合わせ（ハンドシェイク） |
| `capability_response` | Device → Controller | 機能申告（ハンドシェイク） |
| `robot_status` | Robot → Controller | ロボット状態通知（拡張） |

---



## 4.1 接続ハンドシェイク

接続確立後、コントローラーは運用コマンドを送信する前に以下のハンドシェイクシーケンスを必ず実行しなければなりません（MUST）。

```
Controller                      Device
    |                               |
    |-- capability_request -------> |
    |<-- capability_response ------ |
    |                               |
    |-- command(set_time) --------> |
    |<-- response ----------------> |
    |                               |
    |-- command(set_telemetry_interval) -->|
    |<-- response ----------------> |
    |                               |
    |   （ここからテレメトリ開始）    |
```

1. **capability_request / capability_response** — デバイスのサポート内容を問い合わせ
2. **command(set_time)** — デバイスの時刻を同期（RTCなしデバイスに必須）。同期のため任意のタイミングで使用可能
3. **command(set_telemetry_interval)** — テレメトリ送信間隔を指定。デバイスはこれを受信してから初めてテレメトリの送信を開始

---

## 5. profile オブジェクト定義

`profile` はUZCPにおける中心的なデータオブジェクトです。焙煎レシピ（ステップ・制御パラメータ）と焙煎実績（計測値・ログ）を統合的に扱います。

**設計原則:**
- `profile` キーはどのメッセージタイプに現れても**常に同じ構造定義**に従います
- 各コンテキスト（`set_profile`、`roast_log` 等）で必要なフィールドだけを使います
- 未知のフィールドは無視してかまいません（MUST NOT reject）
- これにより `profile` オブジェクトは抽象的・汎用的に扱えます

### 5.1 profile フィールド定義（全フィールド）

#### コアフィールド（常に必須）

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `id` | string | プロファイルID（一意の識別子） |
| `name` | string | プロファイル名（人間が読める表示名） |
| `version` | string | プロファイルのバージョン（例: `"1.0"`） |

#### レシピフィールド（`set_profile` / `get_profile` で使用）

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `steps` | array | 焙煎ステップの配列（下記参照） |
| `charge_temp_et` | float | 投入タイミングの目標排気温度（℃） |
| `charge_weight_g` | float | 投入重量（g） |
| `fc_auto_detect` | bool | 1ハゼの自動検知を使用するか |
| `drop_bt` | float | 排出トリガーとなる豆温度（℃） |
| `drop_ror` | float | 排出トリガーとなる昇温速度（℃/min） |

**steps フィールド定義（各ステップ）:**

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `time` | int | ✅ | 焙煎開始からの経過秒数 |
| `bt_target` | float | — | 目標豆温度（℃） |
| `heater` | int | — | ヒーター出力（0〜100%） |
| `airflow` | int | — | 風量（0〜100%） |
| `drum_rpm` | int | — | ドラム回転数（rpm） |
| `event` | string | — | このステップで発火させるイベント（例: `"drop"`） |

#### 実績フィールド（`roast_log` で使用）

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `title` | string | 焙煎タイトル（任意メモ） |
| `memo` | string | 焙煎メモ（任意） |
| `recorded_at` | string | 焙煎終了時刻（ISO 8601形式） |
| `charge` | object | 投入時実績（下記参照） |
| `drop` | object | 排出時実績（下記参照） |
| `phases` | object | 各フェーズの経過秒数（下記参照） |
| `events` | array | 焙煎中のイベントログ（下記参照） |
| `series` | array | 時系列計測データ（下記参照） |

**charge / drop フィールド:**

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `weight_g` | float | 重量（g） |
| `bt` | float | 豆温度実績（℃） |
| `et` | float | 排気温度実績（℃） |
| `time` | int | （dropのみ）焙煎開始からの経過秒数 |

**phases フィールド:**

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `charge` | int | 生豆投入時の経過秒数 |
| `dry_end` | int | 乾燥終了時の経過秒数 |
| `fc_start` | int | 1ハゼ開始時の経過秒数（`null` = 未発生） |
| `fc_end` | int | 1ハゼ終了時の経過秒数（`null` = 未計測） |
| `sc_start` | int | 2ハゼ開始時の経過秒数（`null` = 未発生） |
| `drop` | int | 排出時の経過秒数 |

**events フィールド（各エントリ）:**

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `time` | int | ✅ | 経過秒数 |
| `type` | string | ✅ | イベント種別（例: `"fc"`, `"airflow"`, `"heater"`, `"drop"`） |
| `value` | float\|null | — | 値（airflow変更値など。イベントによっては `null`） |
| `trigger` | string | — | `"manual"` / `"auto"`（FCなど手動記録か自動検知かを明示） |

**series フィールド（各エントリ）:**

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `time` | int | ✅ | 経過秒数 |
| `bt` | float | — | 豆温度（℃） |
| `et` | float | — | 排気温度（℃）（対応デバイスのみ） |
| `ror` | float | — | 昇温速度（℃/min）（対応デバイスのみ） |

> **補足:** `series` は変化のない区間を省略した疎な配列でも、固定間隔の密な配列でもかまいません（MAY）。受信側は `time` フィールドを基準に補間・プロットしてください。

### 5.2 コンテキスト別の使用フィールド早見表

| フィールドグループ | `set_profile` | `get_profile` (response) | `roast_log` |
| --- | --- | --- | --- |
| コア（id, name, version） | ✅ 必須 | ✅ 必須 | ✅ 必須 |
| レシピ（steps, drop_bt 等） | ✅ 必須 | ✅ 必須 | ✅ 推奨（焙煎時のレシピ記録として） |
| 実績（series, phases 等） | — 無視 | — 無視 | ✅ 必須 |

### 5.3 profile の拡張例（YAGNI — 必要になったら追加）

`profile` は未知のフィールドを無視する設計（MUST NOT reject）なので、
フィールドの追加は既存の実装を壊しません。
以下は将来追加が想定されるフィールドの例です。

#### 外観チェック（焙煎後の豆の状態記録）

```json
"appearance": {
  "oily":      false,
  "uneven":    false,
  "tipping":   false,
  "scorching": false,
  "quakers":   0
}
```

#### カッピング（焙煎後に追記）

```json
"cupping": {
  "notes": "フルーティーでクリーン。アフターに甘さ。",
  "scores": {
    "acidity":    8.0,
    "sweetness":  7.5,
    "body":       7.0,
    "aftertaste": 7.5,
    "balance":    8.0
  }
}
```

#### 環境条件（再現性向上のための記録）

```json
"ambient": {
  "temp_c":      22.0,
  "humidity_rh": 55,
  "altitude_m":  30
}
```

#### series への制御値追加（対応デバイスのみ）

`series` の各エントリは `bt`・`et`・`ror` 以外のフィールドも自由に追加できます。
`cols` 宣言なしにフィールドを追加するだけで済みます。

```json
"series": [
  { "time": 0,   "bt": 80.1, "et": 198.2, "heater": 100, "airflow": 50 },
  { "time": 60,  "bt": 95.3, "et": 201.4, "heater": 90,  "airflow": 50 },
  { "time": 180, "bt": 130.2,"et": 208.1, "heater": 80,  "airflow": 60 }
]
```

> **実装ガイド:** 受信側は知らないフィールドを無視してください（MUST）。
> これにより送信側がフィールドを追加しても、旧バージョンの受信側は壊れません。

---

## 6. メッセージタイプ詳細

### 6.1 telemetry（Device → Controller）

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
    "time": 360,
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
| `time` | int | 秒 | 焙煎開始からの経過秒数 |
| `phase` | string | — | 現在の焙煎フェーズ（後述） |
| `drum_rpm` | int | rpm | ドラム回転数（対応デバイスのみ） |
| `airflow` | int | % | 風量（0〜100） |
| `heater` | int | % | ヒーター出力（0〜100） |
| `fc_detected` | bool | — | 1ハゼ検知フラグ |
| `sc_detected` | bool | — | 2ハゼ検知フラグ |

**フェーズ定義:**

| phase | 必須/任意 | 説明 |
| --- | --- | --- |
| `idle` | ✅ MUST | 待機中（次の焙煎準備完了） |
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
| `stop` | 任意 | 緊急停止中（人間の介入が必要） |

> `idle` と `roasting` 以外のサポートフェーズは `capability_response` で申告しなければなりません（MUST）。

---

### 6.2 command（Controller → Device）

デバイスに制御コマンドを送信します。すべての command に対して、Device は `response` メッセージで応答しなければなりません（MUST）。

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
| `stop` | — | 緊急停止。デバイスはヒーター・ドラムを即時停止し `response` を返す。停止完了後に `stop` または `idle` フェーズへ遷移 |
| `drop` | — | 豆排出トリガー |
| `set_heater` | `{ "value": 0–100 }` | ヒーター出力設定（%） |
| `set_airflow` | `{ "value": 0–100 }` | 風量設定（%） |
| `set_drum_rpm` | `{ "value": 0–200 }` | ドラム回転数設定 |
| `set_profile` | `{ "profile": {...} }` | プロファイル書き込み・更新 |
| `get_profile` | — | プロファイル取得。Device は `response` で `profile` キーを返却 |
| `get_status` | — | デバイスステータス取得。Device は `response` で `device` キーを返却 |
| `get_data` | `{ "fields": ["bt", "et"] }` | 任意データ取得。Device は `response` で `data` キーを返却 |
| `pause` | — | 焙煎一時停止 |
| `resume` | — | 焙煎再開 |
| `set_time` | `{ "ts": 1700000000.0, "tolerance_ms": 1000 }` | デバイスの時刻同期（いつでも使用可能） |
| `set_telemetry_interval` | `{ "interval_ms": 1000 }` | テレメトリ送信間隔設定 |

#### set_profile の例

```json
{
  "uzcp": "1.0",
  "type": "command",
  "id": "cmd010",
  "ts": 1700002000.0,
  "src": "controller-01",
  "dst": "roaster-01",
  "cmd": "set_profile",
  "params": {
    "profile": {
      "id": "ethiopia-natural-light",
      "name": "Ethiopia Natural Light Roast",
      "version": "1.0",
      "charge_temp_et": 200,
      "charge_weight_g": 200,
      "steps": [
        { "time": 0,   "bt_target": 80,  "heater": 100, "airflow": 50, "drum_rpm": 60 },
        { "time": 60,  "bt_target": 110, "heater": 90,  "airflow": 50, "drum_rpm": 60 },
        { "time": 180, "bt_target": 150, "heater": 80,  "airflow": 60, "drum_rpm": 60 },
        { "time": 300, "bt_target": 175, "heater": 70,  "airflow": 70, "drum_rpm": 60 },
        { "time": 420, "bt_target": 195, "heater": 60,  "airflow": 80, "drum_rpm": 60 },
        { "time": 480, "bt_target": 200, "heater": 50,  "airflow": 90, "drum_rpm": 60, "event": "drop" }
      ],
      "fc_auto_detect": true,
      "drop_bt": 200,
      "drop_ror": 3.0
    }
  }
}
```

---

### 6.3 response（Device → Controller）

コマンドの受信確認と、リクエストに対するデータを返します。

```json
{
  "uzcp": "1.0",
  "type": "response",
  "id": "c3d4e5f6",
  "ts": 1700000065.1,
  "src": "roaster-01",
  "dst": "controller-01",
  "ref_id": "b2c3d4e5",
  "status": "ok"
}
```

| フィールド | 説明 |
| --- | --- |
| `ref_id` | 参照している command の id |
| `status` | `ok` / `error` / `unsupported` |
| `error_code` | エラーコード（status が `error` の場合。7章参照） |
| `message` | エラー詳細（status が `error` の場合） |
| `profile` | （`get_profile` の応答時）プロファイルオブジェクト（5章定義） |
| `device` | （`get_status` の応答時）デバイス情報オブジェクト |
| `data` | （`get_data` の応答時）任意のデータオブジェクト |

**status 値:**

| status | 説明 |
| --- | --- |
| `ok` | コマンド受理 |
| `error` | エラー発生（`message` に詳細） |
| `unsupported` | このデバイスでは未サポートのコマンド |

#### get_status の response 例

```json
{
  "uzcp": "1.0",
  "type": "response",
  "id": "r001",
  "ts": 1700001000.0,
  "src": "roaster-01",
  "dst": "controller-01",
  "ref_id": "cmd001",
  "status": "ok",
  "device": {
    "id": "roaster-01",
    "name": "UZU ROASTER UZ-01",
    "firmware": "3.1.0",
    "uzcp_version": "1.0",
    "state": "idle",
    "uptime": 3600
  }
}
```

#### get_profile の response 例

```json
{
  "uzcp": "1.0",
  "type": "response",
  "id": "r002",
  "ts": 1700001010.0,
  "src": "roaster-01",
  "dst": "controller-01",
  "ref_id": "cmd002",
  "status": "ok",
  "profile": {
    "id": "ethiopia-natural-light",
    "name": "Ethiopia Natural Light Roast",
    "version": "1.0",
    "charge_temp_et": 200,
    "charge_weight_g": 200,
    "steps": [
      { "time": 0,   "bt_target": 80,  "heater": 100, "airflow": 50, "drum_rpm": 60 },
      { "time": 480, "bt_target": 200, "heater": 50,  "airflow": 90, "drum_rpm": 60, "event": "drop" }
    ],
    "fc_auto_detect": true,
    "drop_bt": 200,
    "drop_ror": 3.0
  }
}
```

---

### 6.4 event（Device → Controller）

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
    "time": 370,
    "trigger": "manual"
  }
}
```

**イベント一覧:**

| event | 説明 |
| --- | --- |
| `charge` | 生豆投入 |
| `fc` | 1ハゼ検知。`data.trigger` で `"manual"` / `"auto"` を明示 |
| `sc` | 2ハゼ検知。`data.trigger` で `"manual"` / `"auto"` を明示 |
| `drop` | 豆排出 |
| `alarm` | 異常検知（非致命的な警告）。`data.reason` フィールドで原因を示します |
| `emergency_stop` | デバイス側による緊急停止。`data.reason` フィールドで原因を示します |
| `profile_end` | プロファイルシーケンス完了 |
| `cooling_done` | 冷却サイクル完了 |

> **緊急停止について:** コントローラー → デバイスへの停止指示は `command: stop`、デバイス側からの自発的な緊急停止通知は `event: emergency_stop` と非対称になっています。

---

### 6.5 roast_log（双方向 / ファイル保存）

焙煎終了後に焙煎結果を記録・転送します。

**設計原則:**
- エンベロープ（`uzcp`, `type`, `id`, `ts`, `src`, `dst`）は通信用フィールドです
- `profile` オブジェクトがコンテンツ本体です
- **ファイル保存時はエンベロープを除いた `profile` オブジェクトのみを保存します**
- エンベロープと `profile` の一部フィールドが重複してもかまいません（例: タイムスタンプ）

```json
{
  "uzcp": "1.0",
  "type": "roast_log",
  "id": "msg-20231115-001",
  "ts": 1700002416.0,
  "src": "roaster-01",
  "dst": "controller-01",

  "profile": {
    "id": "log-20231115-ethiopia",
    "name": "Ethiopia Natural Light Roast",
    "version": "1.0",
    "title": "コロンビア・スプレモ（中深煎り）",
    "memo": "",
    "recorded_at": "2023-11-15T10:48:00Z",

    "steps": [
      { "time": 0,   "bt_target": 80,  "heater": 100, "airflow": 50, "drum_rpm": 60 },
      { "time": 480, "bt_target": 200, "heater": 50,  "airflow": 90, "drum_rpm": 60, "event": "drop" }
    ],
    "fc_auto_detect": true,
    "drop_bt": 200,
    "drop_ror": 3.0,

    "charge": { "weight_g": 200,   "bt": 178.2, "et": 198.5 },
    "drop":   { "weight_g": 174.5, "bt": 200.3, "et": 221.0, "time": 416 },

    "phases": {
      "charge":   138,
      "dry_end":  246,
      "fc_start": 388,
      "fc_end":   null,
      "sc_start": null,
      "drop":     416
    },

    "events": [
      { "time": 1,   "type": "airflow", "value": 3.0 },
      { "time": 90,  "type": "airflow", "value": 3.7 },
      { "time": 388, "type": "fc",      "value": null, "trigger": "manual" },
      { "time": 416, "type": "drop",    "value": null }
    ],

    "series": [
      { "time": 0,   "bt": 80.1,  "et": 198.2 },
      { "time": 2,   "bt": 82.3,  "et": 199.0 },
      { "time": 416, "bt": 200.3, "et": 221.0 }
    ]
  }
}
```

**ファイル保存形式（`profile` のみ）:**

```json
{
  "id": "log-20231115-ethiopia",
  "name": "Ethiopia Natural Light Roast",
  "version": "1.0",
  "title": "コロンビア・スプレモ（中深煎り）",
  "memo": "",
  "recorded_at": "2023-11-15T10:48:00Z",
  "charge": { "weight_g": 200,   "bt": 178.2, "et": 198.5 },
  "drop":   { "weight_g": 174.5, "bt": 200.3, "et": 221.0, "time": 416 },
  "phases": { "charge": 138, "dry_end": 246, "fc_start": 388, "fc_end": null, "sc_start": null, "drop": 416 },
  "events": [ ... ],
  "series": [ ... ]
}
```

---

### 6.6 capability_request / capability_response

コントローラーがデバイスのサポート機能を問い合わせます。接続確立直後、他のコマンド送信前に必ず実行しなければなりません（MUST）。

**capability_request（Controller → Device）:**

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

**capability_response（Device → Controller）:**

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
    "commands": ["start", "stop", "set_heater", "set_airflow", "set_time", "set_telemetry_interval", "get_status", "get_profile", "set_profile"],
    "events": ["fc", "sc", "drop", "alarm", "emergency_stop"],
    "telemetry_fields": ["bt", "et", "ror", "time", "phase", "heater", "airflow"],
    "phases": ["idle", "roasting", "drying", "maillard", "fc", "development", "drop", "cooling"],
    "roast_log": true,
    "telemetry_interval_min_ms": 100
  }
}
```





| フィールド | 説明 |
| --- | --- |
| `commands` | サポートする `cmd` 値の一覧 |
| `events` | サポートする `event` 値の一覧 |
| `telemetry_fields` | `telemetry.data` に含まれるフィールドの一覧 |
| `phases` | サポートする `phase` 値の一覧（`idle` と `roasting` を含むこと） |
| `roast_log` | `roast_log` メッセージタイプのサポート有無 |
| `telemetry_interval_min_ms` | デバイスが対応できる最小テレメトリ送信間隔（ミリ秒） |

(※)`telemetry_interval_min_ms` が0の場合はテレメトリを返しません

**探索と特定についての指針:**
- **探索時**: コントローラーは `dst: "*"` を使用して、ネットワーク内の全デバイスの情報を一括取得できます
- **特定時**: すでにIDが判明している場合や、複数のUZCPシステムが混在する環境では、特定の `dst` を指定します

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
  "type": "response",
  "id": "r-err-001",
  "ts": 1700000000.0,
  "src": "roaster-01",
  "dst": "controller-01",
  "ref_id": "cmd-001",
  "status": "error",
  "error_code": "E003",
  "message": "heater value out of range: 150"
}
```

---

## 8. ロボット焙煎拡張

完全自動化・ロボット焙煎のための拡張仕様です。

### 8.1 robot_status（Robot → Controller）

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

### 8.2 ロボットコマンド

| cmd | params | 説明 |
| --- | --- | --- |
| `charge` | `{ "weight_g": 200 }` | 指定重量の生豆を投入 |
| `collect` | `{ "bin": "light" }` | 指定ビンに焙煎豆を回収 |
| `clean` | — | ドラムクリーニングシーケンス実行 |
| `standby` | — | ロボットをスタンバイ位置に移動 |

---

## 9. 拡張フィールド

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

## 10. 適合要件

* **MUST（必須）**: `telemetry`、`command`、`response`、`capability_request`、`capability_response`
* **MUST（必須）コマンド**: `set_time`、`set_telemetry_interval`、`get_status`
* **MUST（必須）フェーズ**: `idle`、`roasting`
* **SHOULD（推奨）**: `event`、`set_profile`、`get_profile`、`roast_log`
* **MAY（任意）**: ロボット拡張、`get_data`、`x_` カスタムフィールド

UZCP 1.0 適合を謳うデバイスは、MUSTレベルのメッセージタイプおよびコマンドをすべて実装しなければなりません。

---

## 11. バージョニング

| バージョン | 内容 |
| --- | --- |
| 1.0.0 | 初回リリース。テレメトリ・コマンド・プロファイル・イベント・ロボット拡張を含む |
| 1.0.1 | `set_time`、`set_telemetry_interval`、`capability_request/response` を追加。フェーズ必須要件（`idle`・`roasting`）を明確化。`x_` プレフィックス昇格ポリシーを正式化 |
| 1.0.2 | トランスポート層メッセージデリミタ仕様を追加（3章）。全メッセージの `\n` 終端を必須化。JSONのminify必須化。シリアル・BLE実装例を追加。`capability_response` に `telemetry_interval_min_ms` を追加。デバイス起因の緊急停止用 `emergency_stop` イベントを追加 |
| 1.0.3 | `ack` メッセージを廃止し、すべての command に対する応答を `response` に統一。`profile` メッセージを廃止し、`set_profile` / `get_profile` を command に統合（response で profile キーを返却）。`get_status` / `get_data` コマンドを追加。`set_time` をいつでも使用可能に明記。`capability_request/response` を特別なハンドシェイクメッセージとして明確化 |
| 1.0.4 | **profile オブジェクト統合設計を導入**。`profile` を全メッセージタイプ共通の単一データオブジェクトとして定義（5章）。`roast_log` メッセージタイプを追加（6.5節）。`event` の `fc`/`sc` に `trigger` フィールドを追加。HTTP REST の `GET /command` を `POST /command` に修正。`capability_response` に `roast_log` サポートフラグを追加。エラーコード定義を7章に一本化 |
| 1.0.5 | **時間フィールドを `time` に統一**。`steps[].elapsed` → `time`、`events[].t` → `time`、`events[].v` → `value`、`series[].t` → `time`、`drop.elapsed_sec` → `time`、`telemetry.data.elapsed` → `time`、`event.data.elapsed` → `time`。`ts` をUNIXタイムスタンプから `set_time` 基準の経過秒数に変更。5.3節（profile拡張例）を追加 |

---

## 12. コントリビューション

UZCPはオープンプロトコルです。新しいメッセージタイプ、コマンド、拡張に関する提案はGitHubリポジトリで歓迎します。

* 仕様リポジトリ: [github.com/uzuuzuhonpo/uzcp](https://github.com/uzuuzuhonpo/uzcp)
* Issues & Discussions: コミュニティに公開

---

*UZCP 1.0.5 (Draft) — Designed by Uzuuzu Coffee Roastery*  
*UZU ROASTER（UZU-01）は世界初のUZCP対応デバイスです。*

# 遠隔操作Webサイト API仕様書

**バージョン**: 1.2  
**最終更新日**: 2025/09/15

---

## 更新履歴

| バージョン | 日付 | 内容 |
| :---- | :---- | :---- |
| v1.2 | 2025/09/15 | `isOnline` のリアルタイム状態管理を明確化。`POST /api/stats/compare` エンドポイント追加。`update:status` Socket.IOイベント追加。 |
| v1.1 | 2025/09/01 | `deviceId` のサーバー側自動採番、ROI設定APIの追加、Socket.IOイベントの整理。 |

---

## 1. はじめに

このドキュメントは、「遠隔操作用Webサイト」のフロントエンドとバックエンド間で通信を行うためのAPI仕様を定義するものです。

---

## 2. 共通仕様

| 項目 | 仕様 |
| :---- | :---- |
| **ベースURL** | `/api` |
| **データフォーマット** | すべてのリクエスト・レスポンスのボディは JSON 形式 |
| **認証** | ログインAPI (`/api/auth/login`) を除く全リクエストは、HTTPヘッダーに `Authorization: Bearer <アクセストークン>` を含めること |

---

## 3. エラーレスポンス

APIリクエストが失敗した場合、以下の共通フォーマットでレスポンスを返します。

```json
{
  "message": "エラーの概要を説明するメッセージ",
  "errorCode": "エラーを識別するためのユニークなコード"
}
```

| ステータスコード | 説明 | エラーコード例 | メッセージ例 |
| :---- | :---- | :---- | :---- |
| 400 Bad Request | リクエストが不正（必須パラメータ不足など） | `VALIDATION_ERROR` | "デバイス名は必須です。" |
| 401 Unauthorized | 認証トークンがない、または無効 | `AUTH_REQUIRED` | "認証が必要です。" |
| 403 Forbidden | 認証済みだが、リソースへのアクセス権がない | `FORBIDDEN` | "この操作を行う権限がありません。" |
| 404 Not Found | 指定されたリソースが見つからない | `NOT_FOUND` | "指定されたデバイスが見つかりません。" |
| 500 Internal Server Error | サーバー内部で予期せぬエラーが発生 | `INTERNAL_ERROR` | "サーバーエラーが発生しました。" |

---

## 4. 認証API

### 4.1. ログイン

ユーザーを認証し、アクセストークンを発行します。

- **エンドポイント**: `POST /api/auth/login`

**リクエストボディ**:

```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**レスポンス (200 OK)**:

```json
{
  "accessToken": "ey... (JWT形式のアクセストークン)"
}
```

---

## 5. 機器API（REST API）

### 5.1. 登録機器一覧の取得

- **エンドポイント**: `GET /api/devices`
- **説明**: 認証済みユーザーが登録している全IoT機器の要約リストを取得します。`isOnline` はサーバーが管理するリアルタイムの状態を返します。

**レスポンス (200 OK)**:

```json
[
  {
    "deviceId": "robo-000000",
    "deviceName": "ぼくの正拳突き人形",
    "deviceType": "punching_doll",
    "isOnline": true,
    "summaryData": {
      "label": "今日の正拳回数",
      "value": 150
    }
  }
]
```

### 5.2. 機器の新規登録

- **エンドポイント**: `POST /api/devices`
- **説明**: 新しいIoT機器をシステムに登録します。IDはサーバー側で自動採番されます。

**リクエストボディ**:

```json
{
  "deviceName": "リビングの人形",
  "deviceType": "punching_doll"
}
```

**レスポンス (201 Created)**:

```json
{
  "deviceId": "robo-000001",
  "deviceName": "リビングの人形",
  "deviceType": "punching_doll"
}
```

### 5.3. 機器詳細データの取得

- **エンドポイント**: `GET /api/devices/{deviceId}`
- **説明**: 特定の機器1台の詳細情報（ROI設定を含む）を取得します。`isOnline` はサーバーが管理するリアルタイムの状態を返します。

**レスポンス (200 OK)**:

```json
{
  "deviceId": "robo-000000",
  "deviceName": "ぼくの正拳突き人形",
  "isOnline": true,
  "rois": [
    { "device_id": "robo-000000", "rect": [50, 100, 200, 250] }
  ],
  "realtimeStats": {
    "todayCount": 150,
    "totalCount": 2500
  }
}
```

### 5.4. 機器情報の更新

- **エンドポイント**: `PATCH /api/devices/{deviceId}`
- **説明**: 特定の機器の情報（名前など）を更新します。

**リクエストボディ**:

```json
{
  "deviceName": "新しい名前"
}
```

**レスポンス (200 OK)**:

```json
{
  "deviceId": "robo-000000",
  "deviceName": "新しい名前"
}
```

### 5.5. 機器の削除

- **エンドポイント**: `DELETE /api/devices/{deviceId}`
- **レスポンス (204 No Content)**: レスポンスボディなし

### 5.6. 機器の統計履歴の取得

- **エンドポイント**: `GET /api/devices/{deviceId}/stats`
- **説明**: 特定の機器の過去の統計データを取得します。

**クエリパラメータ**:

| パラメータ | 必須 | 値 |
| :---- | :---- | :---- |
| `period` | ✅ | `month` または `year` |

**レスポンス (200 OK)**:

```json
[
  { "date": "2025-07-01", "count": 120 },
  { "date": "2025-07-02", "count": 85 }
]
```

### 5.7. 機器設定の更新

- **エンドポイント**: `PUT /api/devices/{deviceId}/config`
- **説明**: 特定の機器の設定（監視範囲ROIなど）を更新します。

**リクエストボディ**:

```json
{
  "rois": [
    { "device_id": "robo-000000", "rect": [60, 110, 210, 260] }
  ]
}
```

**レスポンス (200 OK)**:

```json
{
  "message": "設定が更新されました。",
  "rois": [...]
}
```

### 5.8. 複数機器の統計履歴の比較取得（新規）

- **エンドポイント**: `POST /api/stats/compare`
- **説明**: 複数の機器の統計データを、グラフでの比較表示用にまとめて取得します。

**リクエストボディ**:

```json
{
  "deviceIds": ["robo-000000", "robo-000001"],
  "period": "month"
}
```

**レスポンス (200 OK)**:

```json
{
  "robo-000000": [
    { "date": "2025-09-01", "count": 10 },
    { "date": "2025-09-02", "count": 15 }
  ],
  "robo-000001": [
    { "date": "2025-09-01", "count": 5 },
    { "date": "2025-09-02", "count": 20 }
  ]
}
```

---

## 6. リアルタイムAPI（Socket.IO）

### 6.1. クライアント → サーバーへのイベント

| イベント名 | 送信元 | データ例 | 説明 |
| :---- | :---- | :---- | :---- |
| `device:register` | RPi | `{"deviceId": "..."}` | RPiが自身のIDを通知し、部屋に参加する |
| `browser:join_room` | Web | `{"deviceId": "..."}` | Webブラウザが特定のデバイスの部屋に参加する |
| `device:log_punch` | RPi | `{"deviceId": "...", "count": 1}` | パンチを検出したことを通知する |
| `stream:frame` | RPi | `{"deviceId": "...", "image": "..."}` | カメラの映像フレームを送信する |
| `command:execute_action` | Web | `{"deviceId": "...", "action": "..."}` | 遠隔操作命令を送信する |

### 6.2. サーバー → クライアントへのイベント

| イベント名 | 送信先 | データ例 | 説明 |
| :---- | :---- | :---- | :---- |
| `update:stats` | Web | `{"deviceId": "...", "todayCount": ...}` | 統計データの更新をWebブラウザに通知する |
| `update:status` | Web | `{"deviceId": "...", "isOnline": true}` | **（新規）** 機器のオンライン/オフライン状態の変化を通知する |
| `stream:frame` | Web | `{"deviceId": "...", "image": "..."}` | RPiからの映像フレームをWebブラウザに中継する |
| `command:execute_action` | RPi | `{"deviceId": "...", "action": "..."}` | Webからの遠隔操作命令をRPiに中継する |
| `config:update` | RPi | `{"rois": [...]}` | 機器の設定更新をRPiに通知する |

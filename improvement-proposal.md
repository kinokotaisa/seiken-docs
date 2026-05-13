# seikenシステム 改善提案・AWS移行準備メモ

**作成日**: 2026-05-14
**対象リポジトリ**: `seiken-frontend` / `seiken-server` / `seiken-pi-client`

本書は、seikenシステムの現状コードを精査した上で、アーキテクチャ・技術スタック・セキュリティ上の改善点、AWS移行前にローカル完結型のまま修正しておくべき項目、および「AWS実装時のNginx + Gunicorn構成でカメラ映像が表示されない不具合」の原因分析をまとめたものである。

---

## 1. カメラ映像が映らない不具合（Nginx + Gunicorn構成）

複数の原因が重なっている可能性が高い。最大の原因は **ワーカー間でSocket.IOルームが共有されない問題**。

### 1.1. 原因①【最有力】Gunicornマルチワーカー × Socket.IOルーム機能の非対応

現状のコード（`app.py:38-39`）はワーカー内メモリで状態を持つ。

```python
online_devices = set()
sid_to_device = {}
```

Gunicornはデフォルトで複数ワーカープロセスを起動するため、以下が発生する。

```
ラズパイ ──(WebSocket)──► [Worker A] : device:register → ルーム"robo-000000"参加
ブラウザ ──(WebSocket)──► [Worker B] : browser:join_room → ルーム"robo-000000"参加
```

ラズパイから来た `stream:frame` を Worker A が `socketio.emit(..., room='robo-000000')` しても、**Worker A のメモリ上のルーム情報にブラウザは登録されていないため映像が届かない**。

パンチ操作（`update:stats`）も同じ構造だが、たまたまブラウザとラズパイが同じワーカーに繋がれば動くため「ボタンは効くけど映像だけ出ない」のような中途半端な挙動を見せやすい。

**解決策**: Redisをmessage queueとして導入し、ワーカー間でイベントを共有する。

```python
socketio = SocketIO(
    app,
    async_mode='eventlet',  # or 'gevent'
    message_queue='redis://localhost:6379/0',
    cors_allowed_origins=os.getenv('CORS_ORIGINS', '*').split(',')
)
```

さらに `online_devices` / `sid_to_device` も **Redis化**して全ワーカーから参照可能にする必要がある。

### 1.2. 原因②【次点】Gunicornのワーカークラス不適合

Gunicornのデフォルト `sync` ワーカーは WebSocket をサポートしない。長時間ポーリング（HTTP Long Polling）にフォールバックして繋がっているように見えても、30秒で切断ループする可能性が高い。

```bash
# 必須：eventletまたはgevent workerで起動する
gunicorn -k eventlet -w 1 --timeout 0 app:app
```

ワーカー数を増やす場合は必ず原因①の `message_queue` を導入してから。

### 1.3. 原因③ Nginxの長時間接続設定不足

`system-overview.md` のNginx設定には `proxy_read_timeout` / `proxy_send_timeout` の指定がない。`/socket.io` ロケーションには以下が必須。

```nginx
location /socket.io {
    proxy_pass http://app_server;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 86400s;   # ★追加
    proxy_send_timeout 86400s;   # ★追加
    proxy_buffering off;          # ★ストリーミング用
}
```

特に `proxy_buffering off` が無いとフレームがバッファに溜まって遅延・喪失する可能性がある。

### 1.4. 原因④ CORSの固定IP

`app.py:25-31` は `http://192.168.11.33` をハードコード。AWS環境ではCORS拒否されてSocket.IOのハンドシェイク（初期はpolling）が即座に失敗する。

### 1.5. 原因⑤ ペイロード過大（潜在的問題）

`vision_analyzer.py:59-61` で 30fps × Base64エンコードJPEGをSocket.IOで流している。

- 640×480 JPEG ≈ 30〜80KB × 30fps ≈ **1〜2.4MB/秒**
- Base64で33%増（バイナリ送信に変更すれば即削減）
- AWS転送量で月数十〜数百GBになる ⇒ **コスト問題に直結**

---

## 2. アーキテクチャ・設計上の改善点

### 2.1. Socket.IOの認証が形だけ

`socketService.js:16-20` は `auth: { token }` でJWTを渡しているが、サーバー側（`app.py:349-351`）の `handle_connect` は **検証していない**。

```python
@socketio.on('connect')
def handle_connect():
    print(f'✅ クライアントが接続しました: {request.sid}')  # 認証なし
```

任意のクライアントが `browser:join_room` で他人のデバイスIDを指定すれば、**カメラ映像を覗き見できる**セキュリティホール。

```python
@socketio.on('connect')
def handle_connect(auth):
    token = (auth or {}).get('token')
    try:
        payload = jwt.decode(token, app.config['SECRET_KEY'], algorithms=["HS256"])
        # request.sid に user_id を紐付け
    except Exception:
        return False  # 接続を拒否
```

`browser:join_room` 内でも「`device_id` の所有者が `current_user` か」を必ず検証する。

### 2.2. ROI設定APIの所有権検証漏れ

`PUT /api/devices/{id}/config` のリクエストbody内 `rois[].device_id` は検証なしで保存される（`app.py:270-286`）。URL内 `device_id` の所有者であっても、body内に**別ユーザーの所有する `device_id` を含めて保存可能**。Socket.IOで他人の機器のROIを書き換えられる可能性。

### 2.3. オンライン状態管理がメモリ揮発

サーバー再起動・複数ワーカー化で全機器が一旦オフラインに見える。Redis移行と同時に解決する。

### 2.4. シングルカメラ前提のIoTクライアント

`main.py:38` のコメントどおり「最初のカメラ設定のみ使用」。設計仕様書は複数カメラ対応を謳っているがコード未実装。Pi 5は2カメラ対応なので、`VisionAnalyzer` をループ生成 + スレッド配列にリファクタが必要。

### 2.5. フロントエンドの状態管理がチグハグ

- `axios.js:14` で `localStorage.getItem('token')` を毎回直読み
- 一方 `socketService.js:18` は `authStore.state.token`
- ソースが二重化している。**`authStore` 一本に統一**すべき。

### 2.6. 残骸ファイル

- `seiken-server/appold.py` — 不要な旧版
- `seiken-frontend/src/Appbu.vue` — タイポ残骸
- `seiken-frontend/src/components/HelloWorld.vue` — Vite初期ファイル、`App.vue:2` で空import
- `seiken-pi-client/old/` — 動作版含む旧コード（リファレンス用なら `docs/` 等に分離）

---

## 3. 技術スタック見直し

| 項目 | 現状 | 推奨 |
| :--- | :--- | :--- |
| WSGIサーバー | Flask標準（debug=True, allow_unsafe_werkzeug=True）| Gunicorn -k eventlet（本番） |
| 非同期モデル | threading | eventlet または gevent（Flask-SocketIO公式推奨） |
| メッセージブローカー | なし | Redis（AWSなら ElastiCache） |
| DB | SQLite（本番運用中）| PostgreSQL（RDS）。スキーマは現状のままで `DATABASE_URL` 切替のみ |
| 映像伝送 | Base64 JPEG over Socket.IO 30fps | **WebRTC**（P2P）または MediaMTX/Janus + HLS。少なくともバイナリJPEG送信に変更 |
| JWT | コードに `"default_super_secret_key"` フォールバック | 環境変数必須化、起動時チェック |
| eventlet | requirements.txtに有るが未使用 | 整理（使うなら gevent も検討。eventlet は Python 3.12+で互換性懸念） |
| シークレット管理 | `.env` ファイル | AWS Secrets Manager / SSM Parameter Store |

---

## 4. AWS移行前にローカルで直しておくべきこと

優先度順：

1. **【必須】環境変数化** — `SERVER_URL`, `CORS_ORIGINS`, `DATABASE_URL`, `SECRET_KEY`。コード内のIPハードコード（`config.py:43`, `app.py:25-31`）を全削除
2. **【必須】Socket.IO認証の実装** — connect時のJWT検証 + `browser:join_room` で所有権チェック
3. **【必須】Redis導入 + message_queue動作確認** — Docker Composeでローカル検証。これがAWSでの映像不具合の核心解決
4. **【必須】Gunicorn + eventlet で動作確認** — ローカルで `gunicorn -k eventlet --workers 2 app:app` を試験。映像が流れる条件を固める
5. **【必須】`debug=True` / `allow_unsafe_werkzeug=True` の除去** — 本番に絶対残してはいけない
6. **【強く推奨】Nginx設定に `proxy_read_timeout` / `proxy_buffering off`** を追加してローカル本番構成として固定
7. **【強く推奨】映像をバイナリJPEG送信に変更** — base64削除で帯域33%削減。さらに余力があればWebRTC化検討
8. **【推奨】ROI config APIの所有権検証強化**
9. **【推奨】`appold.py` / `Appbu.vue` / 旧コード削除**
10. **【推奨】authStoreとaxiosのトークン取得統一**
11. **【任意】複数カメラ対応のリファクタ**
12. **【任意】PostgreSQL対応のローカル検証**（Docker ComposeでPostgresを立てて `DATABASE_URL` 切替試験）

---

## 5. AWS構成案（参考）

```
Route 53
   │
   ▼
CloudFront (HTTPS, SPAキャッシュ)
   │
   ▼
ALB (sticky session 必須: Socket.IOのため)
   │
   ▼
ECS Fargate (Gunicorn + eventlet, 1ワーカー/コンテナ × N台)
   │           │
   │           ├─► ElastiCache Redis (Socket.IO message_queue + online状態)
   │           │
   │           └─► RDS PostgreSQL
   │
   └─► S3 (フロントエンド静的ファイル)

ラズパイ (現地) ──WebSocket──► ALB (TLS) ──► ECS
```

- **ALBの sticky session 設定が必須**（Socket.IOの初期ポーリングが同じワーカーに行く必要）
- ECSコンテナは「1ワーカー/コンテナ」にして横スケールはコンテナ数で行う
- 映像のみ別配信パス（MediaMTX/Kinesis Video Streams）に逃がすとコスト・遅延が大きく改善

---

## 6. 結論

最も影響度の高いのは **「Redis message_queue + eventlet worker + ALB sticky session」の3点セット**。これが現状のAWS映像不具合の核である。

ローカル完結型のうちに上記の構成（Redis + Gunicorn eventlet + Nginx本番設定）で動作確認を完了させてから AWS へ移行することで、移行時のトラブルを最小化できる。
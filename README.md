# seiken-docs

**seikenシステム**の設計仕様書・技術ドキュメントを管理するリポジトリです。

---

## seikenシステムとは

子供向けおもちゃ（正拳突き人形）とIT・電子工作技術を組み合わせた、**遠隔操作Webアプリシステム**です。

スマホやPCのブラウザから、ローカルネットワーク越しに物理デバイスをリアルタイムで操作できます。カメラ映像のストリーミング、パンチ回数の自動カウント、統計グラフの可視化なども備えています。

> 子供がテクノロジーに興味を持つきっかけを作ることを目的としたプロジェクトです。
> 開発プロセスはYouTube動画・Zenn記事としてコンテンツ発信しています。

---

## システム構成

1台のRaspberry Pi 5がサーバー・クライアント・Webホストの3役を兼任します。

```
スマホ/PCブラウザ
       ↓ HTTP (ポート80)
  Nginx (リバースプロキシ)
       ↓
  Flask APIサーバー ←→ SQLite DB
       ↑
  IoTクライアント (Python)
    ├── カメラ映像解析 (OpenCV)
    └── GPIO制御 → 正拳突き人形
```

**技術スタック**

| レイヤー | 技術 |
| :--- | :--- |
| フロントエンド | Vue.js 3 / Vite |
| バックエンド | Python / Flask / Flask-SocketIO |
| データベース | SQLite |
| IoTクライアント | Python / picamera2 / OpenCV / gpiozero |
| インフラ | Raspberry Pi 5 / Nginx / systemd |

---

## リポジトリ一覧

| リポジトリ | 説明 |
| :--- | :--- |
| [seiken-server](https://github.com/kinokotaisa/seiken-server) | FlaskバックエンドAPIサーバー |
| [seiken-frontend](https://github.com/kinokotaisa/seiken-frontend) | Vue.js フロントエンド（WebUI） |
| [seiken-pi-client](https://github.com/kinokotaisa/seiken-pi-client) | Raspberry Pi IoTクライアント |
| [seiken-avr-firmware](https://github.com/kinokotaisa/seiken-avr-firmware) | AVRマイコン用ファームウェア |
| [seiken-docs](https://github.com/kinokotaisa/seiken-docs) | 設計仕様書・ドキュメント（本リポジトリ） |

---

## ドキュメント一覧

| ドキュメント | 説明 | 最終更新 |
| :--- | :--- | :--- |
| [system-overview.md](./system-overview.md) | システム全体構成・セットアップ手順 | 2025/11/10 |
| [api-spec.md](./api-spec.md) | REST API・Socket.IO イベント仕様 | 2026/05/11 |
| [backend.md](./backend.md) | バックエンド設計仕様 | 2026/05/11 |
| [frontend.md](./frontend.md) | フロントエンド設計仕様 | 2025/09/29 |
| [rpi-client.md](./rpi-client.md) | Raspberry Pi クライアント設計仕様 | 2025/09/01 |

---

## ライセンス

本リポジトリのドキュメントは個人プロジェクトのものです。参考にする場合はクレジットを記載してください。
# Prospector Scanner + Dongle Mode

[t-ogura/zmk-config-prospector](https://github.com/t-ogura/zmk-config-prospector) をベースにした ZMK キーボード用ステータスディスプレイ＋USBドングルプロジェクトです。

オリジナルのプロジェクトに感謝します。素晴らしいハードウェア設計とスキャナーモードのUI実装をベースに、ドングルモード（BLE Central → USB HID転送）と複数キーボード管理機能を追加しました。

---

## オリジナルからの変更点

### ドングルモード（BLE Central + USB HID転送）

オリジナルのスキャナーモード（BLEアドバタイズを受信して表示のみ）に加えて、**ドングルモード**を追加しました。

```
┌─────────────┐  BLE (HID over GATT)  ┌──────────────┐  USB HID  ┌──────┐
│  キーボード  │ ──────────────────────→│  Prospector  │──────────→│  PC  │
│  (ZMK)      │                        │  (ドングル)   │           │      │
└─────────────┘                        └──────────────┘           └──────┘
```

- Prospector が BLE Central として ZMK キーボードに接続
- HID レポート（キーボード・マウス・コンシューマー）を USB HID で PC に転送
- GATT 経由でリアルタイムにキーボードステータスをディスプレイに表示

### 複数キーボード管理

- 最大3台（設定で最大5台）のキーボードとボンディング可能
- タッチUIでキーボード切替、新規ペアリング、削除
- ボンド情報は NVS に永続化（再起動後も保持）

### その他の変更

- BLE LESC ペアリング対応（PSA/MbedTLS経由のECC）
- GATT MTU 65 で 26 バイトステータス通知対応
- USB CDC ACM シリアルコンソール（デバッグ用）

---

## ビルドバリアント

| バリアント | artifact-name | 説明 |
|-----------|---------------|------|
| スキャナー | `prospector_scanner` | ディスプレイのみ（BLEアドバタイズ受信） |
| スキャナー + タッチ | `prospector_scanner_touch` | スワイプ操作 + 設定画面 |
| ドングル | `prospector_scanner_dongle` | USB HID転送 + ディスプレイ |
| ドングル + タッチ | `prospector_scanner_dongle_touch` | USB HID転送 + スワイプ操作 |

---

## ドングルモードの使い方

### ドングル側の設定

`config/prospector_scanner_dongle.conf` で主要な設定を行います：

```conf
CONFIG_PROSPECTOR_DONGLE_MODE=y

# ターゲットキーボード名（プレフィックス一致、空=最初のHIDデバイス）
CONFIG_PROSPECTOR_DONGLE_TARGET_NAME="lotom"

# 複数キーボード対応
CONFIG_BT_MAX_PAIRED=3
CONFIG_PROSPECTOR_DONGLE_MAX_BONDED=3  # UIに表示する最大台数（1-5）
```

### キーボード側の対応

ドングルモードでキーボードを接続するために、キーボード側に必要な設定を説明します。

#### 1. BLE デバイス名の設定

ドングルはBLEデバイス名のプレフィックスマッチでキーボードを検出します。キーボードの `.conf` で名前を設定してください：

```conf
# キーボードの .conf ファイル
CONFIG_ZMK_KEYBOARD_NAME="lotom"
```

ドングル側の `CONFIG_PROSPECTOR_DONGLE_TARGET_NAME="lotom"` と一致させてください。プレフィックスマッチなので、`"lotom"` は `"lotom"`, `"lotom_left"`, `"lotom_right"` すべてにマッチします。

#### 2. ペアリング前の settings_reset

キーボードが以前別のデバイスとボンディングしていた場合、古いボンド情報が残っているとドングルからの接続がタイムアウトします（`Connection creation timeout` エラー）。

**初回ペアリング前に、キーボード側で settings_reset を行ってください：**

1. `settings_reset` ファームウェアをキーボードにフラッシュ
   ```
   # ZMK の settings_reset シールドを使用
   west build -b your_board -- -DSHIELD=settings_reset
   ```
2. フラッシュ後、通常のキーボードファームウェアを再フラッシュ
3. キーボードのボンドテーブルがクリアされ、新規ペアリング可能に

> **settings_reset が必要なケース：**
> - ドングルとの初回ペアリング時
> - ドングルの BLE ID が変わった場合（完全リフラッシュ後など）
> - ドングルのログに `Connection creation timeout` が繰り返し表示される場合
>
> **不要なケース：**
> - 既にペアリング済みのキーボード間の切替
> - 電源サイクル後の通常再接続

#### 3. Prospector Status GATT サービス（任意）

ドングルのディスプレイにキーボードのステータス（バッテリー、レイヤー、WPM等）をリアルタイム表示するには、キーボード側にも Prospector モジュールが必要です。

キーボードの `config/west.yml` に追加：

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: prospector
      url-base: https://github.com/t-ogura

  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml

    - name: prospector-zmk-module
      remote: prospector
      revision: v2.1.0
      path: modules/prospector-zmk-module
```

キーボードの `.conf` に追加：

```conf
CONFIG_ZMK_STATUS_ADVERTISEMENT=y
CONFIG_ZMK_STATUS_ADV_KEYBOARD_NAME="lotom"
```

> このモジュールがなくてもドングルとしての USB キーボード転送は動作します。ディスプレイにステータスが表示されないだけです。

### ペアリング手順

1. キーボード選択画面に移動（メイン画面から上スワイプ）
2. **「+ Add New」** をタップ
3. キーボードの電源が入っていること、他のホストに接続されていないことを確認
4. 数秒以内に発見されたキーボードが一覧に表示される
5. **キーボード名をタップ**してペアリング開始
6. ペアリング成功後、ボンド済みリストに追加される
7. 再起動後も自動的に接続

### キーボード選択画面の操作

```
┌──────────────────────────────────┐
│ Keyboards                        │
├──────────────────────────────────┤
│ ● lotom          [Connected]     │  タップ → 接続切替
│ ○ KobitoKey                      │  タップ → 接続切替
│                                  │
│ [+ Add New]                      │  タップ → ペアリングモード
│                                  │
│                   ↓ Main         │  下スワイプ → メイン画面に戻る
└──────────────────────────────────┘
```

- **タップ**: そのキーボードに接続切替
- **長押し**: 削除確認ダイアログ表示
- **「+ Add New」**: 新しいキーボードのペアリングモードに入る

---

## 参考リンク

- **[オリジナル Prospector Scanner](https://github.com/t-ogura/zmk-config-prospector)** — ベースとなったプロジェクト（スキャナーモードの詳細なドキュメント、ハードウェア情報、設定ガイドはこちらを参照）
- **[オリジナル Prospector ハードウェア](https://github.com/carrefinho/prospector)** — carrefinho によるハードウェア設計
- **[Prospector キット](https://shop.beekeeb.com/products/zmk-wireless-dongle-prospector-diy-kit)** — beekeeb で購入可能
- **[ZMK Firmware](https://github.com/zmkfirmware/zmk)**
- **[タッチモードガイド](docs/TOUCH_MODE.md)** — タッチパネルの設定と操作方法

---

## ライセンス

MIT License。詳細は `LICENSE` ファイルを参照。

本プロジェクトは以下のオープンソースプロジェクトを利用・参考にしています：

- **[t-ogura/zmk-config-prospector](https://github.com/t-ogura/zmk-config-prospector)** — スキャナーモードUI、ハードウェア設計（MIT License）
- **[carrefinho/prospector](https://github.com/carrefinho/prospector)** — Prospector ハードウェアプラットフォーム（MIT License）
- **[ZMK Firmware](https://github.com/zmkfirmware/zmk)** — ZMK キーボードファームウェア（MIT License）
- **[YADS](https://github.com/janpfischer/zmk-dongle-screen)** — UIウィジェットデザイン（MIT License）

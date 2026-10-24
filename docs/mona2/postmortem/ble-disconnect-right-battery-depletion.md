---
title: moNa2 の BLE 切断は右手の電池切れが原因だった
id: mona2-ble-disconnect-0001
type: postmortem
category: mona2
status: active
created: 2026-10-25
valid_as_of: 2026-10-25
owner: "@rtosshy"
tags: [mona2, ble, battery, macos, zmk]
---

> ✅ **ACTIVE（有効・2026-10-25 時点）** — これは点時刻の記録です。以降の状況変化はこの記録を無効化しません。最新の決定は後継チェーンを辿ってください。

# moNa2 の BLE 切断は右手の電池切れが原因だった

## 要約

moNa2 が Mac から勝手に切れた原因は、右手側（split central）の電池切れだった。無線やハードの異常ではない。根本原因は、**電源スイッチ OFF のまま充電していた**ため、電池にほとんど充電されていなかったこと。

## 症状

- 2026-10-24〜25 に、Mac との BLE 接続が何度も切れた（10/24 だけで約10回、10/25 08:03 に1回）。
- 切断時のログはすべて同じだった。

  ```
  BTLEServer: Peripheral "<private>" is disconnected: Error Domain=CBErrorDomain Code=6
  "The connection has timed out unexpectedly."
  ```

  `Code=6` は BLE の supervision timeout（HCI 0x08）で、Mac が一定時間、右手側からのパケットを1つも受け取れなかったことを示す。

## 原因

右手側の残量（`0x0106`）と切断時刻が一致していた。

| 時刻 | 右手側の残量 | 切断 |
|---|---|---|
| 10/24 10:29 | 3% | 10:33 timeout |
| 10/24 11:17〜 | 0% | 16:15〜19:58 に timeout 7回 |
| 10/25 08:01 | 0% | 08:03 timeout |

切断は右手側の残量が 0〜3% の時間帯にしか起きていない。左手側（`0x0108`）は 76〜84% で、ほとんど減っていなかった。

**根本原因：スイッチ OFF で充電していた。** 公式ファームの README に「電源をONにした状態でのみ充電」とある。スイッチは電池と XIAO の充電回路（BAT 端子）の間にあるので、OFF だとケーブルを挿しても電池は充電されない。ユーザーが「充電していた」と認識していた時間帯の多くは、ログに充電の記録がなかった。

## 検証

右手のスイッチを OFF にして電池切れを再現し、Mac ではない USB 充電器につないだ。

| 時刻 | 出来事 |
|---|---|
| 08:37:13 | スイッチ OFF → `Code=6` timeout で切断（本番の切断と同じ記録） |
| 08:37:25 | 充電器の給電で起動 → 12秒後に BLE で再接続。左手側との通信も復活 |

Mac の USB 機器一覧に moNa2 は出ていなかったので、BLE だけで再接続している。ここからわかること：

- 電池切れは、今回の切断とまったく同じ記録を残す。
- 復帰に必要なのは電気だけで、Mac との USB 通信は要らない。
- スイッチは電池の線だけを切っている（スイッチ OFF でも USB 給電で起動する）。

## 誤診と教訓

- **左手の残量を右手のものと取り違えた。** 切断直前に届いた「77%」は左手側の値だった。Mac には左右2つの残量が届くので、どちらの値かを確認してから判断する（見分け方は [電池残量の確認と充電](../how-to/battery-check-and-charging.md)）。
- **`DiagnosticControl` のログは切断の記録ではない。** `audioaccessoryd` の `DiagnosticControl` は `TriggerReason: "Anomaly"`・`DisconnectReason: 0` の、HID の遅延が大きいときの統計レポート。切断を追うなら `BTLEServer` の `is disconnected` を見る。
- **`d2d8b12`（`CONFIG_BT_CTLR_PHY_2M=n`）は誤診にもとづく変更。** 無線の問題を疑って入れたが、変更後に切断が減ったように見えたのは、同じ時間帯に少し充電していたから。効果の根拠はない。2M PHY のほうが通信時間が短く電池にわずかに有利なので、`f9a343b` でリバートした。
- 送信出力アップ（`CONFIG_BT_CTLR_TX_PWR_PLUS_8`）や peripheral latency の変更も検討したが、原因が電池だったので不要。

## 対応

- 充電は**スイッチ ON にしてから**ケーブルを挿す。満タンになると自動で止まる。
- 右手側が 30% を切ったら充電する（理由は how-to を参照）。

## 関連

- [電池残量の確認と充電](../how-to/battery-check-and-charging.md)
- [config/mona2_r.conf](../../../config/mona2_r.conf)
- [sayu-hub/zmk-config-moNa2（公式ファームの README）](https://github.com/sayu-hub/zmk-config-moNa2)

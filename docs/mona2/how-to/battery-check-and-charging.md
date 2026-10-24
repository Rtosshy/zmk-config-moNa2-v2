---
title: moNa2 の電池残量の確認と充電
id: mona2-battery-howto-0001
type: how-to
category: mona2
status: active
created: 2026-10-25
last_reviewed: 2026-10-25
review_due: 2027-04-23
owner: "@rtosshy"
tags: [mona2, battery, macos, zmk]
---

> ✅ **ACTIVE（有効）** — 最終レビュー: 2026-10-25 ／ 次回レビュー期限: 2027-04-23

# moNa2 の電池残量の確認と充電

## 要約

右手側（split central）は Mac との接続・左手との接続・トラックボールを全部担当するので、左手側よりずっと早く減る。右手側が切れると Mac との接続ごと切れる。残量は macOS のログで左右別に確認でき、充電は**スイッチ ON** で行う。

## 充電する

1. 電源スイッチを **ON** にする（OFF だと電池が切り離されていて充電されない）
2. USB-C ケーブルを挿す。Mac でも USB 充電器でもよい
3. 満タンになると自動で止まるので、挿しっぱなしでよい。ON のままなので充電中も使える

電池は左右それぞれ約 170mAh。XIAO の充電電流は小さい（初期値で約 50mA）ので、空からなら満充電まで数時間かかる。10〜20分ではほとんど入らない。

## 残量を確認する

**Mac の Bluetooth メニューやバッテリーウィジェットの数字は使わない。** 左右の区別なく1つの数字しか出ず、左手側（ほとんど減らない）の値が出ていることがある。それを見て「まだ余裕がある」と判断すると、右手側の電池切れを見逃す。

macOS の `ioreg` や `system_profiler` でも moNa2 の残量は取れない。`BTLEServer` のログに約1分おきに左右別で記録されているので、それを読む。

```sh
/usr/bin/log show --last 1h --info \
  --predicate 'process == "BTLEServer" AND eventMessage CONTAINS "Battery level"' \
  --style compact | tail -6
```

```
... Battery level for peripheral "<private>" (0x0106): 31%   ← 右手
... Battery level for peripheral "<private>" (0x0108): 77%   ← 左手
```

- fish では `log` が別のコマンドに解決されることがあるので、`/usr/bin/log` と絶対パスで書く。
- `0x0106` / `0x0108` は Mac 内部の識別番号で、ラベルではない。ペアリングをやり直すと変わる可能性がある。

### どちらが右手か確かめる

片方だけをスイッチ ON で充電して数分待ち、上のコマンドで**値が増えたほう**がその半分。2026-10-25 時点では `0x0106` = 右手、`0x0108` = 左手だった。

## 残量 % の読み方

ZMK の % は、電圧を 3.45V〜4.2V の範囲で直線的に割り振っただけの値。リチウム電池の電圧は残量に比例しない（中盤は粘り、空になる直前に急落する）ので、% は時間に対してまっすぐ減らない。

- **30% を切ったら充電する。** そこから先は表示よりずっと早く減る。実例：右手側が 24% のまま19時間動かず、そのあと1時間で 10%、4時間で 0% になった。
- **充電中の % はあてにしない。** 電圧が上がるので高めに出る（挿した瞬間に 95% と表示され、抜いたら 17% に戻った例がある）。抜いて少し待った後の値を見る。
- ±1〜2% の行き来（9% ↔ 10% など）は、負荷による電圧の揺れ。

## 電池切れで止まったとき

スイッチ ON のまま、USB 充電器・モバイルバッテリー・Mac のどれかにつなぐ。USB の給電だけで起動し、十数秒で BLE が再接続する（Mac との USB 通信は要らない）。

## 関連

- [moNa2 の BLE 切断は右手の電池切れが原因だった](../postmortem/ble-disconnect-right-battery-depletion.md)
- [config/mona2_r.conf](../../../config/mona2_r.conf)（`CONFIG_RGBLED_WIDGET_BATTERY_LEVEL_*` で LED の警告しきい値を変えられる）
